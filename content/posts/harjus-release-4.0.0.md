---
title: "Harjus v4 adds kernel bypass and more"
subtitle: "Harjus gets 10% faster tick-to-trade but remains unprofitable"
date: 2026-03-03T21:06:01+02:00
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

After releasing the triangular arbitrage trading bot [Harjus](https://github.com/ValtteriL/harjus) under MIT license and publishing [a write-up on my journey]({{< relref "binance-triangular-arbitrage" >}}) last year, I took a brief stint of working on other stuff.
However, I was constantly feeling that I had given up too early, when there were still major optimizations that could be enough to turn a profit.

My previous write-up had caught the eye of multiple aspirational arbitrageurs who, to my pleasure, contacted me in search of advice.
Trying hard not to give any false hope, I still had to note that there just might be a chance of turning a profit with optimized Harjus as a one-man show.
What followed was always the hard question of why I wasn't working on it.

Finally, I had to give in to the temptation and look under this stone.
The result is [release 4.0.0](https://github.com/ValtteriL/harjus/tree/releases/4.0.0), which employs the key optimizations and clearly beats earlier versions.
This post presents its highlights and the results I got trading with it.

## Notable changes

The single optimization identified earlier that would speed up Harjus the most was [bypassing the kernel](https://blog.cloudflare.com/kernel-bypass/).
Ie, embedding a network stack into it and using it to control the NIC on the host without help from the Linux kernel.

I decided to implement this by using [F-Stack](https://f-stack.org/) by Tencent.
F-Stack greatly simplifies the kernel bypass by providing a TCP stack out of the box, along with a high-level API, and facilities for pinning CPU cores.
[The F-Stack repo](https://github.com/F-Stack/f-stack) also contains a microthreading API, which I decided to utilize for additional benefits.

The implementation I did by forking [Quickfix Engine](https://quickfixengine.org/), stripping it of unused features, and adding an initiator that uses F-Stack.
The fork is called FlashFix, and it's [part of the Harjus repo](https://github.com/ValtteriL/harjus/tree/releases/4.0.0/flashfix).

Because of the low-level nature of F-Stack, I could no longer use Nix as the development environment.
I couldn't use my workstation directly either, as my NIC was not supported.
Instead, I had to switch to Conan for managing libraries and recreate the development environment with VMs using Vagrant.

### Other goodies

Finding the availability zone closest to the Binance servers was a recurring problem many faced.
I therefore added a utility and [instructions](https://github.com/ValtteriL/harjus/blob/releases/4.0.0/docs/choose-optimal-az.md) for automating the required measurements.

{{< figure src="/images/harjus-release-4.0.0/harjus-v4-diagram.svg" alt="High level diagram of Harjus deployment in AWS" caption="High level diagram of Harjus deployment in AWS" >}}

## Latency improvement

To measure how v4 compares to the previous version (3.2.0), I created a simple FIX server that measures the tick-to-trade latency from its clients.
I then ran both versions against it and collected the results.

The raw results for 1000 samples were as follows:

| Measure     | v3.2.0  | v4.0.0  |  Change |
| :---------- | :------ | :------ | ------: |
| Avg (µs)    | 1165.97 | 1047.65 | -118.32 |
| Median (µs) | 1144.35 | 1008.74 | -135.61 |
| Min (µs)    | 956.00  | 873.67  |  -82,33 |
| Max (µs)    | 3836.29 | 3834.48 |   -1.81 |

Harjus v4 beats the previous version clearly on all measures.
On average, it has 10% (0.1ms) lower tick-to-trade latency.
The difference is so large that the previous version would go out of business, everything else being equal.

It is interesting how the latency distribution stays similar on both versions.
Bypassing the kernel is supposed to offer predictable performance, which led me to expect a lot lower spread.
This can be caused by my imperfect measurement setup (the FIX server used a regular kernel-provided TCP/IP stack).

## Results

I ran Harjus in production for 74 hours between Feb 27th and Mar 1st.
When starting, I had $100 BNB, $250 in BTC, and $600 in USDC.
My commission percentage was 0.075% (the lowest Binance tier + BNB discount).

I focused on 273 assets, which formed 4500 potential triangular arbitrage paths.
The paths consisted of 665 different trading pairs. The focused assets were as follows:

```ini
ASSETS="A,A2Z,AAVE,ACH,ACT,ACX,ADA,AEVO,AIXBT,ALGO,ALLO,ALT,ANIME,APE,API3,APT,AR,ARB,ARKM,ASTER,AT,ATOM,AUCTION,AVAX,AVNT,AXS,BABY,BANANA,BANK,BARD,BB,BCH,BEAMX,BERA,BIGTIME,BIO,BLUR,BMT,BNB,BOME,BONK,BREV,BTC,C,CAKE,CATI,CETUS,CFX,CGTP,CHESS,CHZ,CKB,COMP,COOKIE,COTI,COW,CRV,CVC,CVX,CYBER,DASH,DF,DOGE,DOGS,DOLO,DOT,DYDX,DYM,EDEN,EGLD,EIGEN,ENA,ENJ,ENS,ENSO,EPIC,ERA,ETC,ETH,ETHFI,EUL,EUR,EURI,F,FET,FF,FIL,FLOKI,FLUX,FOGO,FORM,FUN,GALA,GIGGLE,GMT,GMX,GPS,GRT,GUN,HAEDAL,HBAR,HEI,HEMI,HIVE,HMSTR,HOLO,HOME,HUMA,HYPER,ICP,IDEX,ILV,IMX,INIT,INJ,IO,IOTA,JTO,JUP,JUV,KAIA,KAITO,KERNEL,KITE,KMNO,LA,LAYER,LDO,LINEA,LINK,LISTA,LPT,LSK,LTC,LUNA,LUNC,MAGIC,MANTA,MASK,MAV,ME,MEME,MET,MINA,MIRA,MITO,MMT,MORPHO,MOVE,MUBARAK,NEAR,NEIRO,NEO,NEWT,NIL,NMR,NOM,NOT,NXPC,OG,OM,ONDO,ONT,OP,OPEN,ORCA,ORDI,OSMO,PARTI,PENDLE,PENGU,PEOPLE,PEPE,PHA,PIXEL,PLUME,PNUT,POL,PROVE,PUMP,PUNDIX,PYTH,QNT,QTUM,RARE,RAY,RED,RENDER,RESOLV,REZ,ROSE,RPL,RSR,RUNE,RVN,S,SAGA,SAHARA,SAND,SAPIEN,SEI,SENT,SHELL,SHIB,SIGN,SKL,SKY,SNX,SOL,SOLV,SOMI,SOPH,SPK,SSV,STEEM,STO,STRK,STX,SUI,SUSHI,SXT,SYN,SYRUP,T,TAO,THE,THETA,TIA,TLM,TNSR,TON,TOWNS,TRB,TREE,TRUMP,TRX,TST,TURBO,TURTLE,TUT,TWT,UMA,UNI,USDC,USUAL,UTK,VANA,VANRY,VELODROME,VET,VIRTUAL,W,WAL,WBTC,WCT,WIF,WLD,WLFI,XAI,XLM,XPL,XRP,XTZ,XVG,YB,YGG,ZBT,ZEC,ZEN,ZK,ZKC,ZKP,ZRO"
```

The experiment resulted in a total of 84 opportunities with a total profit of 6.07788e-05 BTC ($4,18), out of which none were successfully executed.
All of the attempts failed due to order expiry.
18 executions resulted in some trades being filled.
24 trades were filled in total.

Afterwards, my assets had been reallocated.
I was holding $600 in ADA, $235 in PENDLE, $100 in FET, $15 in TRX, $10 in APT, and finally $7 in BTC.
{{< figure src="/images/harjus-release-4.0.0/binance-balance-after-running-trading-bot.png" alt="Balance at the end of trading session" caption="Balance at the end of trading session" >}}

See [the raw log of the deployment](https://gist.github.com/ValtteriL/b438a17c3c0eb61d4000ce93e9f1e39c).

## Why the poor results?

Why are all executions failing in expiry?
Why are there only $4.18 available to be arbed out, when last year there were $176,35?

The first is because either the orders are cancelled by the other party, or someone else fills them before I do.
In short, someone is being faster.

The second is because the EU protects my money from crypto scams.

As an EU citizen, I am unable to trade the de facto casino chip (USDT) on Binance.
USDT has the highest trading volume globally, and trading it leaves out a lot of opportunities.

While I was busy with the v4, [Binance also removed TRY](https://www.binance.com/en/square/post/10-30-2025-binance-to-implement-changes-for-turkish-lira-trading-pairs-31714427466017) (Turkish Lira) from Binance.com.
TRY [contained]({{< relref "binance-triangular-arbitrage#results-2" >}}) the most arbitrage opportunities in 2025 after BTC and USDC.
This means I have even fewer arbitrage opportunities to exploit.

The number of opportunities varies daily, and I might have gotten unlucky, too.
The competition may also have gotten better.

### How can someone be faster?

I identified possible reasons [already before]({{< relref "binance-triangular-arbitrage#why-is-it-not-profitable" >}}), namely, having higher Binance VIP tier, using bare metal servers, magic AWS network optimizations, or knowledge of exploitable quirks in Binance.

Thanks to my shoestring budget, I deploy Harjus on a virtual machine where I pay the overhead tax and suffer unpredictability caused by noisy neighbours.
Deploying on bare metal would offer higher and more predictable performance.
I'm deploying on a c6in.xlarge instance, which costs $0.2856 per hour on-demand.
Someone [more serious](https://github.com/janestreet/magic-trace/wiki/Supported-platforms,-programming-languages,-and-runtimes#supported-virtualization-environments) outside the EU could choose, for instance, c6i.metal, paying $6.848 per hour on-demand and still clear a profit.

As a brand new reason, Binance introduced the [Simple Binary Encoding (SBE)](https://developers.binance.com/docs/binance-spot-api-docs/fix-api#fix-sbe) in its FIX API.
SBE is far superior to the standard FIX encoding when it comes to latency.
The effect should be a lot less than what kernel bypass offers vs. the regular kernel network stack.
Still, Harjus does not use it and is thus at a disadvantage.

## Demo

Exploiting triangular arbitrage opportunities in the Binance testnet with Harjus v4.
[![asciicast](https://asciinema.org/a/789065.svg)](https://asciinema.org/a/789065)

## Conclusion

> Running one of the simplest high-frequency liquidity-taking strategies on the most liquid crypto exchange and turning a profit is very hard. It may be doable by a single person, provided the playing field is level, but it certainly requires hyperfocus. The money awaiting is not earth-staggering, especially without Binance VIP or if the assets traded are limited.

I'll echo that conclusion from the previous writeup. However, I rate the challenge way harder, and the expected value a lot less than I did before.

My plan is to step out of this rabbit hole humbled and work on something else.

Best of luck to any fellow arbitrageurs, and beware: Hope springs eternal.
