---
title: "Harjus: Triangular Arbitrage Bot for Binance"
subtitle: "My journey chasing riskless profits in the world’s most liquid crypto exchange at high speed with a grayling"
date: 2025-07-21T09:14:28+03:00
description: "TODO"
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

{{< figure src="/images/binance-triangular-arbitrage/harjus-logo.png" alt="Harjus logo" >}}

The idea of turning on a program and just letting it rip while collecting (riskless) money is compelling.
No sales or marketing, or other things I, as an engineer, like to shy away from doing - just enjoyable tinkering with models and infra.
That's printing money.

To have a go at money printing, I built a trading bot.
The bot is called Harjus (Finnish for grayling), and it exploits [triangular arbitrage](https://en.wikipedia.org/wiki/Triangular_arbitrage) opportunities within the Binance spot market.

My journey from ignorance and naivety to fluency in finance- and crypto bro speak and a working bot was quite long and complicated, even though the trading idea is trivial.
Ultimately, I was not able to make the bot profitable.

This post is a deep dive into the bot's implementation, the process of how the building process went, what I tried, what the problems were, and what I think is required to make the bot profitable.
Hope it speeds up your development and strategy selection efforts, as well as reveals possible sources of alpha.

Harjus is MIT licensed and [available on GitHub](https://github.com/ValtteriL/harjus).

## Background

I dipped my toe into practical algorithmic trading after my friend told me about his other friend who makes a living arbitraging crypto markets.
Understandably, my friend didn't reveal this arbitrageur's operations in detail, but he implied that it was basically [cross-exchange arbitrage](https://www.coinapi.io/learn/glossary/cross-exchange-arbitrage).

To get algotrading experience under my belt, I wanted to get the simplest horse with minimal risk off to the races.
At the time, of such strategies, I could only think of cross-exchange arbitrage and intra-exchange triangular arbitrage.

As I didn't want to spend time integrating with multiple exchanges or threaten the living of my friend's friend, I chose the latter strategy.
For the exchange, I chose Binance, as I already had an account there from way back. I figured trading crypto would be the most straightforward, as the market data is free and APIs are freely accessible to retail traders.

Triangular arbitrage is well known, simple, and the fastest exploiter will eventually eat everyone else.
It also used to be extremely profitable within crypto;
In 2018 there were [14.55% opportunities that lasted over 3 seconds](https://steemit.com/cryptocurrency/@zaccharles/cryptocurrency-triangular-arbitrage) in the Bittrex exchange.

As markets get more efficient, there are fewer and smaller opportunities, and they last for a shorter time.
I'm sure the market efficiency has increased dramatically in the last 8 years, especially in Binance, as it's the market leader.
At the time of writing, [an online arbitrage scanner](https://www.octobot.cloud/tools/triangular-arbitrage-crypto) reported the last opportunity over 1% on Binance 10 months ago.

Even if I were looking at crypto, the other arbitrageurs looking to exploit these same opportunities are not only hobbyists.
Some sophisticated trading firms from the traditional finance, such as Jump Trading and Jane Street, also participate.

Given the hypercompetitiveness, I knew that the bot is extremely latency sensitive, and even if successful, there may not be much juice left.
I still wanted to give it a shot, if nothing else than a learning experience.

Before starting, I recorded all profitable arbitrage opportunities for my fee level among all assets on the Binance Spot market.
In 24h, I got total of 3966 opportunities, which resulted in a profit of $4419,78 if exploited.
Bitcoin had the most opportunities (1292), with a total profit of $229,86, and it required holding only $1279,90 in BTC to be able to capture them.
On Finnish standards, this signaled the existence of a nice payday, so I decided to go ahead with the implementation.

I named the bot Harjus after my favorite fly fishing target, grayling.
Grayling lives in a rapid (the exchange) and has its territory (the universe of trading symbols) where it hunts for small bugs (arbitrage opportunities).
Despite eating only bugs, it can grow quite big.
Grayling's territory can be taken over by a contender (as I was hoping to do to the biggest arbitrageur).

## Basic mode of operation

Throughout all revisions, Harjus works roughly as follows:

1. Get current balances & available symbols
2. Calculate all possible trading paths
3. Subscribe to the best bid & ask of chosen symbols
4. On each bid&ask update, calculate profit for trading paths given balance
5. If a profitable path is detected, send orders for the trades (Limit order at the calculated price, Fill Or Kill)
6. On receipt of execution reports, update the balance

## Naive first attempt (releases 1.0.x)

Surely no serious trader with a manager will risk ending up holding a bag of fartcoin, right?
This intuition of mine led me to believe I have an edge if I just look for opportunities spanning any coin available.
This is a lot of symbols (~3000) to subscribe to and a huge amount of possible trading paths.

I chose to implement the bot using Elixir for its concurrency, robustness, and real-time story.
For ease of deployment, I went with ECS.

When I started developing, the only way to get real-time updates of best bid and offer was [though websocket streams](https://developers.binance.com/docs/binance-spot-api-docs/web-socket-streams#individual-symbol-book-ticker-streams).
Luckily, that was easy in Elixir.
The fastest way to submit trades was [though the FIX API](https://developers.binance.com/docs/binance-spot-api-docs/fix-api#newordersingle-d).
I couldn't afford to compromise on this, so I had to develop a FIX client in Elixir covering just enough of the protocol to make it work.

{{< figure src="/images/binance-triangular-arbitrage/harjus-1-architecture.png" alt="Harjus 1 architecture diagram" caption="Mixed level architecture of the 1.0.x release" >}}

### Network optimization

By placing my deployment server as close to Binance servers as possible and ensuring high-speed connectivity, I could shave the most out of the latency.
Binance's servers were running in AWS Tokyo (ap-northeast-1), so I placed mine there too.

I was interested only in the latency to the Websocket stream and FIX API order entry servers.

For websocket, there were multiple alternatives for hostnames, including `stream.binance.com`, `data-stream.binance.vision`, and `data-stream.binance.com`, as well as ports (443 or 9443).
The hostnames also resolved to multiple IPs.
I attempted to outsmart the resolver by [measuring the latency to all of the alternatives myself and hardcoding the best one to the hosts file](https://github.com/ValtteriL/harjus/blob/e8fc824c6422b6b52e981e250be73397ea33dbb5/deploy/main.tf#L135).

For FIX, there was only a single order entry server and a single port.

Through these shenanigans, I was able to reduce the RTT to the servers to ~1-2ms.
For comparison, the RTT with a websocket server was a staggering 250ms from my home network.

### Results

In the Binance testnet, the arbitrage opportunities were frequent and quite big.
Harjus was able to successfully exploit roughly 7/8 of all detected opportunities, and my testnet balance was skyrocketing.

When deploying to production, however, it could not keep up with market updates at all.
CPUs were constantly at 100%, and the backlog of unprocessed market updates was growing faster than they could be processed.
After running for a while, it would crash when the memory ran out.

Despite implementing all optimizations I could think of, this version did not fly.
Elixir just doesn't seem equipped to deal with such volumes of events that each demand many high-precision calculations.

## Speeding up with ports (releases 1.1.x)

To mitigate the speed issue, I carved out the computationally expensive part (calculating arbitrage opportunities) of the program into [a port](https://hexdocs.pm/elixir/Port.html) that I implemented in C++.
I first considered making a [Native Implemented Function (NIF)](https://www.erlang.org/doc/system/nif.html) out of it instead, but the port seemed much more approachable.

The bad thing with the choice was that I had to format and parse the data passed between Elixir and the C++ programs.
I opted for JSON for simplicity, but it added overhead to the operation.

{{< figure src="/images/binance-triangular-arbitrage/harjus-1-1-architecture.png" alt="Harjus 1.1.x architecture diagram" caption="Mixed level architecture of the 1.1.x release" >}}

### Results

In testnet, Harjus could now exploit ~11/12 of the detected opportunities.

In production, it ran into the same problem as the earlier release, but stayed alive longer.

## Going fully native (releases 2.x.y onwards)

After two half-measures, it was time to be pragmatic.
Therefore, I rewrote the whole application in C++, which is also used by the sophisticated actors.

At this time, Binance had also made it possible to subscribe to market data through the FIX API.
Thus, I was able to cut the websocket part altogether and do all latency-sensitive communication over FIX.

C++ being used by others for the same purpose meant that I was able to use the [Quickfix library](https://github.com/quickfix/quickfix) for FIX communication.
This made FIX comms easy, but the last official release was dated, lacking features, and the available OS package for it in Nix was built without SSL support.
I solved the problem by [patching and packaging it myself](https://github.com/ValtteriL/harjus/blob/c16f033227ebad881a9ef105a8abed009019789b/default.nix#L23).

{{< figure src="/images/binance-triangular-arbitrage/harjus-3-architecture.png" alt="Harjus 3 architecture diagram" caption="Mixed level architecture of the 3.x.y release" >}}

### Optimizations

The possibilities for optimization are endless.
To avoid the deepest rabbit holes, I decided to apply the 80/20 principle and apply only the optimizations with the best ROI.

#### Network

Since previous versions, I learned that network latency within a region could be further reduced by placing communicating workloads into the same availability zone (AZ).

The AZ of Binance's FIX servers is not public knowledge, but I was able to pinpoint them in `ap-northeast-1a` by [measuring the latency](https://github.com/ValtteriL/harjus/blob/releases/3.1.0/docs/network-performance.md) from all three.
Mind you, the zone may be different for you because of ID randomization by AWS, and thus, you should do the measurement yourself.

This trick brought the latency down to ~0.5ms.

```text
Max rtt: 1.009ms | Min rtt: 0.513ms | Avg rtt: 0.552ms
Raw packets sent: 100 (4.000KB) | Rcvd: 100 (4.600KB) | Lost: 0 (0.00%)
Nping done: 1 IP address pinged in 99.21 seconds
```

#### Harjus

Using C++ opened a myriad of micro-optimization possibilities in the application.

I implemented the application lock-free, avoided allocation on the hot path, and compiled it along with the patched Quickfix using [aggressive optimization options](https://github.com/ValtteriL/harjus/blob/c16f033227ebad881a9ef105a8abed009019789b/default.nix#L11) and removed all [performance affecting binary hardening](https://github.com/ValtteriL/harjus/blob/c16f033227ebad881a9ef105a8abed009019789b/default.nix#L17).

I also uncontainerized the application.
It was deployed as a Nix package.

#### Production server

The biggest source of latency after the network was the OS.
Using a general-purpose off-the-shelf OS like AWS Linux 2023 meant that it was not optimized for the workload.

The way to the best network latency would have been to bypass the kernel altogether with DPDK and using a user-space network stack.
This would have required vast changes to the Quickfix library, and thus, I decided to skip that at least for now.

A lesser way to improve performance was to tune the OS.
Marc Richards' [study](https://talawah.io/blog/linux-kernel-vs-dpdk-http-performance-showdown/) shows that, through tuning, one may be looking at only a 51% performance disadvantage to DPDK.
Without optimizations, the disadvantage could reportedly be many times that.

I applied the following optimizations to the OS:

1. RT kernel
2. Busy socket polling
3. Disabling Iptables
4. Disabling interrupt modification and dynamic interrupt moderation
5. Disabling speculative execution mitigations

And others mainly applied by Marc in his post, [recommended by AWS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ena-improve-network-latency-linux.html), or included in the default Tuned profiles for latency and real-time systems.
You can find my [tuned profile](https://github.com/ValtteriL/harjus/blob/releases/3.2.0/deploy/playbooks/files/tuned.conf) here.

### Results

This version was the fastest.
As a simple comparison, the trading path calculation now takes 3 seconds, when in the previous version it took over 40.

In the testnet, Harjus caught 30/31 of the detected opportunities.
As an acid test, I deployed Harjus to the testnet on the production server and left it running for 48 hours.
The result was $60k in profit.

In production, Harjus handled updates from all trading symbols with ease.
However, almost all of its executions failed.
It sometimes succeeded in some of the trades, but at least one of its orders would expire without a fill.
This resulted in unwanted exposure to the asset it happened to end up with, which was invariably fartcoin-tier volatility nightmares.
It turned out to be hard to get rid of the exposure without manual intervention and taking a loss, as arbitrage opportunities in these assets were a lot rarer than in bitcoin.

In an attempt to resolve the exposure problem and further reduce latency, I limited the assets it would trade to a select few at the expense of the total number of opportunities.
These assets were fiat, had high volume at the time, or were stablecoins.
I tried to strike a balance between subjective stability and the number of arbitrage opportunities involving the assets.
This resulted in focusing on [these 26 assets](https://github.com/ValtteriL/harjus/blob/69adb62151f8ab44d793258ad5f7a54182c80ed4/deploy/playbooks/group_vars/harjus_instance/prod.yml#L24), which at the time made up 93 trading symbols and formed 804 different arbitrage paths.

{{< figure src="/images/binance-triangular-arbitrage/triarb-opportunity-count-by-asset-over-10.png" alt="Bar plot of arbitrage opportunity count by asset where count > 10" caption="Assets with most arbitrage opportunities between Jun 12th and 13th 2025" >}}

Running Harjus for 69 hours with the focused set of assets resulted in 156 opportunities, out of which only a single one was successfully executed.
With my commission percentage, the total profit from all these opportunities was $176,35.
The successful execution was worth $0.13.

The opportunities detected are lucrative enough to turn a profit, provided enough of them are captured.
Currently, over 99% were not captured.
Assuming the failed executions result in no loss, this kind of trading profit is not even close to breaking even.

The number of opportunities observed extrapolates to just $1840/month.
That kind of money should not interest the big guns.
However, this is only the arbitrage opportunities that are profitable with my commission fee level (0.075%).
A big-time trader is likely in Binance's VIP program, which can bring the taker fee down to 0.01725%.
This means the opportunities visible to me are merely remnants that the sharks weren't fast enough to capture.

## Why is it not profitable?

Someone is faster.

An arbitrage execution fails when a submitted order expires without a fill.
So either the order previously observed on the book was already filled, or it was cancelled.
In both cases, there is someone faster than us.

They might be faster because they have implemented the optimizations I didn't.
Network-wise, I couldn't think of ways to improve, and apparently, colocation is not available.
An AWS networking guru might still be able to squeeze some latency out.
On the host, the clear path to reduced latency was DPDK and/or a combination of OS tweaks.
An obvious improvement would also be to use a bare metal server instead of a VM.

They could also know of some quirks of the exchange I don't.
One possibility for such a quirk is the sending order of market data updates.
As the updates flow through TCP, everyone gets their updates at different times.
The sending order matters, and thus might be exploitable if not randomized.
I expect at least Binance's trading divisions (Sigma Chain, Merit Peak Limited, possibly others) to be aware of all exploitable quirks, if such exist.

## Demo

Exploiting triangular arbitrage opportunities in the Binance testnet
[![asciicast](https://asciinema.org/a/730934.svg)](https://asciinema.org/a/730934)

## Conclusion

Running one of the simplest high-frequency liquidity-taking strategies on the most liquid crypto exchange and turning a profit is very hard.
It may be doable by a single person, provided the playing field is level, but it certainly requires hyperfocus.
The money awaiting is not earth-staggering, especially without Binance VIP or if the assets traded are limited.

Even though I did not reach the money printing phase, this odyssey of over 1300 commits was not for nothing.
It gave me the excuse to learn a bunch of new things like Elixir, Nix, and C++, and left me with immense interest in and appreciation of financial engineering.

If you intend to try similar arbitrage, I hope my experience either lit lightbulbs in your head or got you to reconsider.
