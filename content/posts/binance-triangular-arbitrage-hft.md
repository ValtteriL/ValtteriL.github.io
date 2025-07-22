---
title: "Harjus: Triangular Arbitrage Bot for Binance"
subtitle: "TODO"
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

The idea of turning on a program and just letting it rip while collecting (riskless) money is compelling.
No sales or marketing, or other things I as an engineer like to shy away from doing - just enjoyable tinkering with models and infra.
That's literally printing money.

To have a go at money printing, I built a trading bot.
The bot is called Harjus (Finnish for crayling) and it exploits [triangular arbitrage](https://en.wikipedia.org/wiki/Triangular_arbitrage) opportunities within the Binance spot market.

The journey from ignorance and naivety to fluency in finance- and crypto bro speak and a working bot was quite long and complicated even though the trading idea is extremely simple.
Ultimately, I was not able to make the trading bot profitable.

This post is a deepish dive into the bot's implementation, the process of how the building process went, what I tried, what were the problems, and what I think is required to make the bot profitable.
Hope it speeds up your development and strategy selection efforts, as well as reveals possible sources of alpha.

Harjus is MIT licensed and [available on GitHub](https://github.com/ValtteriL/harjus).

## Background

I dipped my toe into practical algorithmic trading after my friend told me about his other friend that makes a living arbitraging crypto markets.
Understandably, my friend didn't reveal this arbitrageour's operations in detail, but he implied that it was basically [cross-exchange arbitrage](https://www.coinapi.io/learn/glossary/cross-exchange-arbitrage).

To get algotrading experience below my belt, I wanted to get the simplest horse with minimal risk off to the races.
At the time, of such strategies I could only think of cross-exchange arbitrage, and intra-exchange triangular arbitrage.

As I didn't want to spend time integrating with multiple exchanges or threaten the living of my friend's friend, I chose the latter strategy.
For the exchange I chose Binance, as I already had an account there from way back. I figured trading crypto would be the most straight forward, as the market data is free and APIs are freely accessible to retail traders.

Triangular arbitrage is well known, simple, and the fastest exploiter will eventually eat everyone else.
It also at least used to be extremely profitable within crypto;
In 2018 there were [14.55% opportunities that lasted over 3 seconds](https://steemit.com/cryptocurrency/@zaccharles/cryptocurrency-triangular-arbitrage) in the Bittrex exchange.

As markets get more efficient, there are fewer and smaller opportunities, and they last shorther time.
I'm sure the market efficiency has increased dramatically in the last 8 years, especially in Binance as it's the market leader.
At the time of writing, [an online arbitrage scanner](over 1%) reported last opportunity over 1% on Binance 10 months ago.

Even if I was looking at crypto, the other arbitrageurs looking to exploit these same opportunities are not only hobbyists.
Some sophisticated trading firms from the traditional finance, such as Jump Trading and Jane Street, also participate.

Given the increasngly efficient market and participant list I knew that the bot is extremely latency sensitive, and even if successful, there may not be much juice left.
I still wanted to give it a shot, if nothing else than a learning experience.

I named the bot Harjus after my favorite fly fishing target, crayling.
Crayling lives in a rapid (the exchange) and has its own territory (the universe of trading symbols) where it hunts for small bugs (arbitrage opportunities).
Despite eating only bugs, it can grow quite big.
Crayling's territory can be taken over by a contender (as I was hoping to do to the biggest arbitrageur).

## Basic mode of operation

Thorough all revisions, Harjus works roughly as follows:

1. Get current balances & available symbols
2. Calculate all possible trading paths
3. Subscribe to the best bid & ask of chosen symbols
4. On each bid&ask update, calculate profit for trading paths given balance
5. If profitable path detected, send orders for the trades (Limit order at the calculated price, Fill Or Kill)
6. On received execution reports, update balance

## Naive first attempt (releases 1.x.y)

Surely no serious trader with a manager will risk ending up holding a bag of fartcoin, right?
This intuition of mine led me to believe I have an edge if I just look for opportunities spanning any coin available.
This is a lot of symbols (~3000) to subscribe to and a huge amount of possible trading paths.

I chose to implement the bot using Elixir for its concurrency, robustness, and real-time story.
For ease of deployment, I went with ECS.

When I started developing, the only way to get real-time updates of best bid and offer was [though websocket streams](https://developers.binance.com/docs/binance-spot-api-docs/web-socket-streams#individual-symbol-book-ticker-streams).
Luckily that was easy in Elixir.
The fastest way to submit trades was [though the FIX API](https://developers.binance.com/docs/binance-spot-api-docs/fix-api#newordersingle-d).
I couldn't afford compromising on this, so I had to develop a FIX client in Elixir covering just enough of the protocol to make it work.

### Network optimization

By placing my deployment server as close to Binance servers as possible and ensuring high-speed connectivity, I could shave the most out of the latency.
Binance's servers were running in AWS Tokyo (ap-northeast-1), thus that's where I placed mine too.

I was interested only about the latency to the Websocket stream and FIX API order entry servers.

For websocket, there were multiple alternatives for hostnames including `stream.binance.com`, `data-stream.binance.vision`, and `data-stream.binance.com`, as well as ports (443 or 9443).
The hostnames also resolved to multiple IPs.
I attempted to outsmart the resolver by [measuring the latency to all of the alternatives myself and hardcoding the best one to hosts file](https://github.com/ValtteriL/harjus/blob/e8fc824c6422b6b52e981e250be73397ea33dbb5/deploy/main.tf#L135).

For FIX, there was only a single order entry server and a single port.

Through these shenaganians I was able to reduce the RTT to the servers to ~1-2ms.
For comparison, the RTT with a websocket server was a staggering 250ms from my home network.

### Results

In Binance testnet the arbitrage oppotunities were frequent and quite big.
Harjus was able to successfully exploit roughly 7/8 of all detected opportunities and my testnet balance was skyrocketing.

When deploying to production however, it could not keep up with market updates at all.
CPUs were constantly at 100%, and the backlog of unprocessed market updates was growing faster than they could be processed.
After running for a while, it would crash when memory ran out.

Despite implementing all optimizations I could think of, this version did not fly.
Elixir just isn't equipped with dealing with such volumes of events that each demand many high-precision calculations.

## Speeding up with ports (releases 2.x.y)

To mitigate the speed issue, I carved out the computationally expensive part (calculating arbitrage opportunities) of the program into [a port](https://hexdocs.pm/elixir/Port.html) that I implemented in C++.
I first considered making a [Native Implemented Function (NIF)](https://www.erlang.org/doc/system/nif.html) out of it instead, but the port seemed much more approachable.

The bad thing with the choice was that I had to format&parse the data passed between elixir and the C++ programs.
I opted for JSON for simplicity, but it added overhead to the operation.

### Results

In testnet, Harjus could now exploit ~11/12 of detected opportunities.

In production, it ran to the same problem as the earlier release, but stayed alive longer.

## Going fully native (releases 3.x.y)

After two half-measures, it was time for a full measure.
Therefore I rewrote the whole application in C++, which is used also by the sophisticated actors.

At this time, Binance had also made it possible to subscribe to market data through the FIX API.
Thus, I was able to cut the websocket part altogether and do all latency-sensitive communication over FIX.
Apart from tidying the codebase, having only a single server to communicate with made latency optimization easier.

C++ being used by others for the same purpose meant that I was able to use the [Quickfix library](https://github.com/quickfix/quickfix) for FIX communication.
This made FIX comms easy, but the last official release was dated, and available OS package for it in nix was built without SSL support.
I solved the problem by [packaging it myself](https://github.com/ValtteriL/harjus/blob/c16f033227ebad881a9ef105a8abed009019789b/default.nix#L23).

### Optimizations

The possibilities for optimizing are endless.
To avoid the deepest rabbit holes, I tried to apply the 80/20 principle and apply only the most effective optimizations.

#### Network

Since I had only a single server to communicate with, I placed my server in the same availability zone.
The availability zone of Binance's servers is not public knowledge, but I was able to pinpoint the one by [measuring the latency](https://github.com/ValtteriL/harjus/blob/releases/3.1.0/docs/network-performance.md) from all three.

This brought the latency down to ~0.5ms.

```text
Max rtt: 1.009ms | Min rtt: 0.513ms | Avg rtt: 0.552ms
Raw packets sent: 100 (4.000KB) | Rcvd: 100 (4.600KB) | Lost: 0 (0.00%)
Nping done: 1 IP address pinged in 99.21 seconds
```

#### Harjus

Using C++ opened a myriad of micro-optimization possibilities in the application.

I implemented the application lock-free, avoided allocation on the hot path, and compiled it along with quickfix using [aggressive optimization options](https://github.com/ValtteriL/harjus/blob/c16f033227ebad881a9ef105a8abed009019789b/default.nix#L11) and removed all [performance affecting binary hardening](https://github.com/ValtteriL/harjus/blob/c16f033227ebad881a9ef105a8abed009019789b/default.nix#L17).

I also uncontainerized the application.
It is now deployed as a Nix package.

Finally, I reduced the number of symbols subscribed to.
This reduced the number of events, and allows operating with fewer threads.

#### Production server

The production server I optimized by

1. RT kernel
2. Tuned profile
3. Ethtool

### Results

In testnet Harjus caught now 30/31 of detected opportunities.
As an acid test I deployed Harjus to testnet on the production server, and left it running for 48 hours.
The result was $60k in profit.

Calculating trading paths took now 3 seconds. The elixir implementation took over 40.

## Why is it not profitable?

Apparently Binance does not offer colocation through cluster placement groups that would make t ...

## Conclusion

### Learnings
