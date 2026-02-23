---
title: "Harjus 4.0.0: kernel bypass and more"
subtitle: "Harjus gets 10% faster with kernel bypass"
date: 2026-02-18T21:06:01+02:00
description: "Harjus 4.0.0 uses kernel bypass to XXXX"
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

The implementation I did by forking Quickfix Engine, stripping it from unused features, and adding an initiator that uses F-Stack.
The fork is called FlashFix, and it's [part of the Harjus repo](https://github.com/ValtteriL/harjus/tree/releases/4.0.0/flashfix).

Because of the low-level nature of F-Stack, I could no longer use Nix as the development environment.
I couldn't use my workstation directly either, as my NIC was not supported.
Instead, I had to recreate the development environment with VMs using Vagrant.

## Other goodies

Finding the availability zone closest to the Binance servers was the ...

###

## Results

Harjus gets 10% faster tick-to-trade with kernel bypass

## Why is it (still) not profitable?

- TRY
- FIX binary encoding

### Callout

## Demo

## Conclusion
