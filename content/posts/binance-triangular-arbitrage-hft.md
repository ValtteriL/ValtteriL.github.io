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

This post is a deepish dive into the process of how the bot building process went, what I tried, what were the problems, and what I think is required to make the bot profitable.
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

In deployment, I noticed that the biggest wins in latency TODO

In Binance testnet the oppotunities were frequent and quite big.
This version was able to successfully exploit roughly 7/8 of all detected opportunities and testnet balance was skyrocketing.

However when deploying to production, it could not keep up with market updates at all.
CPUs were constantly at 100%, and the backlog of unprocessed market updates was growing faster than they could be processed.
After running for a while, it would crash when memory ran out.

### Problems

1. Speed
2. Failing trades

## Speeding up with ports (releases 2.x.y)

## Going native (releases 3.x.y)

## Optimizations

## Conclusion

### Learnings
