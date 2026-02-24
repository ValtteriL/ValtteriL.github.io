---
title: "Harjus v4 adds kernel bypass and more"
subtitle: "Harjus gets 10% faster tick-to-trade but remains unprofitable"
date: 2026-02-18T21:06:01+02:00
description: "Harjus v4 adds kernel bypass for 10% speedup and more"
tags: [
"binance",
"trading",
"hft",
"arbitrage"
]
categories: [
"Trading",
]
draft: false
---

After releasing [Harjus](https://github.com/ValtteriL/harjus) under MIT license and publising [a writeup on my journey](binance-triangular-arbitrage.md) last year, I took a brief stint of working on other stuff.
However, I was constantly feeling that I gave up too early, when there were still major optimizations that could be just enough to turn a profit.

My previous writeup had caught the eye of multiple aspirational arbitrageours who, to my pleasure, contacted me in search of advice.
Trying hard not to give any false hope, I still had to note that there just might be a chance of turning a profit with optimized Harjus as a one man show.
What followed was always the hard question why I wasn't working on it.

Finally, I had to give in to the temptation and look under this stone.
The result is [release 4.0.0](https://github.com/ValtteriL/harjus/tree/releases/4.0.0), which employs the key optimizations and clearly beats earlier versions.
This post presents its highlights, and the results I got trading with it.

TL;DR: I was still unable to make it profitable.
You might still be able to do it (see [callout](#callout)).

## Notable changes

The single optimization identified earlier that would speed up Harjus the most was [bypassing the kernel](https://blog.cloudflare.com/kernel-bypass/).
Ie. embedding a network stack into it and using it to control the NIC on the host without help from the Linux kernel.

I decided to implement this by using [F-Stack](https://f-stack.org/) by Tencent Cloud.
F-Stack greatly simplifies the kernel bypass by providing a TCP stack out of the box, along with high-level API, and facilities for pinning CPU cores.
[The F-Stack repo](https://github.com/F-Stack/f-stack) contains also a microthreading API, which I decided to utilize for additional benefits.

The implementation I did by forking [Quickfix Engine](https://quickfixengine.org/), stripping it from unused features, and adding an initiator that uses F-Stack.
The fork is called FlashFix, and it's [part of the Harjus repo](https://github.com/ValtteriL/harjus/tree/releases/4.0.0/flashfix).

Because of the low-level nature of F-Stack, I could no longer use Nix as the development environment.
I couldn't use my workstation directly either, as my NIC was not supported.
Instead, I had to switch to Conan for managing libraries and recreate the development environment with VMs using Vagrant.

### Other goodies

Finding the availability zone closest to the Binance servers a recurring problem many faced.
I therefore added a utility and [instructions](https://github.com/ValtteriL/harjus/blob/releases/4.0.0/docs/choose-optimal-az.md) for automating the required measurements.

## Results

To measure how v4 compares to the previous version (3.2.0), I created a simple FIX server that measures the tick-to-trade latency from its clients.
I then ran both versions against it and collected the results.

The raw results were as follows:

```text
Version 4.0.0
=== Tick-to-Trade Latency Statistics (1000 samples) ===
  Avg:    1047.65 µs
  Median: 1008.74 µs
  Min:    873.67 µs
  Max:    3834.48 µs
================================================


Version 3.2.0
=== Tick-to-Trade Latency Statistics (1000 samples) ===
  Avg:    1165.97 µs
  Median: 1144.35 µs
  Min:    956.00 µs
  Max:    3836.29 µs
================================================
```

Harjus gets 10% faster tick-to-trade with kernel bypass

## Why is it (still) not profitable?

As an EU citizen I am unable to trade USDT on Binance.
While I was busy with the v4, Binance also restricted me from trading TRY (Turkish Lira).
These two coins [contained the most arbitrage opportunities in 2025](binance-triangular-arbitrage.md#results-2) after BTC.
This means I have a lot fewer trading opportunities.

Binance also introduced the [Simple Binary Encoding SBE](https://developers.binance.com/docs/binance-spot-api-docs/fix-api#fix-sbe) in its FIX API.
SBE is far superior to the standard FIX encoding when it comes to latency.
Harjus does not use it, and is thus at a disadvantage.

### Callout

## Demo

Exploiting triangular arbitrage opportunities in the Binance testnet with Harjus v4.
[![asciicast](https://asciinema.org/a/789065.svg)](https://asciinema.org/a/789065)

## Conclusion

To repeat what I wrote in the first Harjus writeup: Running one of the simplest high-frequency liquidity-taking strategies on the most liquid crypto exchange and turning a profit is very hard.
But unlike what I wrote in

I have again
