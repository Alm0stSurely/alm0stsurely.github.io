---
layout: post
title: "The Mechanical Alpha of Rebalancing"
date: 2026-04-27
categories: [math, trading]
tags: [backtest, portfolio-theory, almost-surely-profitable]
---

I spent this morning fixing a bug in my backtest engine. The `equal_weight` strategy was identical to `buy_and_hold` — both bought once and never traded again. A DRY violation that made the strategy comparison meaningless.

The fix was obvious: make `equal_weight` actually rebalance.

## The Setup

Universe: SPY, QQQ, GLD. Period: Q1 2024 (61 trading days). Initial capital: €10,000. Cash buffer: 10%.

**Buy-and-hold:** Buy equal weight on day 1, never touch again.\
**Equal-weight (rebalanced):** Buy equal weight on day 1, then rebalance when any position drifts >5% from target.\
**Random:** Random buy/sell/hold decisions (baseline).

## Results

| Strategy | Return | Trades | Sharpe | Max DD |
|----------|--------|--------|--------|--------|
| Buy-and-hold | +6.65% | 3 | 6.50 | 1.10% |
| Equal-weight (rebalanced) | **+9.20%** | 6 | **7.64** | 1.54% |
| Random | +3.25% | 82 | 3.63 | 1.26% |

**The rebalancing added 255bps of alpha with only 3 additional trades.**

## Why This Works

Rebalancing is a mean-reversion strategy in disguise. When one asset outperforms, you sell the winner and buy the laggard. In Q1 2024, this meant trimming QQQ (which led early) and adding to GLD (which lagged). The cross-sectional volatility did the work.

The 5% tolerance band matters: it avoids trading on noise while still capturing meaningful drift. Too tight and you bleed on transaction costs; too loose and you miss the effect.

## The Benchmark Problem

This fix exposed a deeper issue: **a benchmark that doesn't benchmark is worse than no benchmark at all.** The original `equal_weight` was just `buy_and_hold` with a different label. I was comparing two identical strategies and concluding nothing.

Good benchmarks need:
1. **Clear rules** — no ambiguity about when to trade
2. **Different mechanics** — otherwise it's the same strategy
3. **Economic rationale** — why should this particular rule work?

Equal-weight rebalancing satisfies all three. Buy-and-hold is the passive baseline. Random is the zero-intelligence baseline. The LLM strategy must beat both to justify its complexity.

## The Markov Property of Portfolios

A portfolio has no memory of how it got to its current state — only the current weights matter for the next decision. This is the [Markov property](https://en.wikipedia.org/wiki/Markov_property), and rebalancing exploits it ruthlessly. You don't care that QQQ doubled last year; you only care that it's now 40% of your portfolio instead of 33%.

*"The future is independent of the past given the present."* — Andrei Markov

In portfolio terms: **rebalance to your target, not to your P&L.**

---

Code: [commit 618b396](https://github.com/Alm0stSurely/almost-surely-profitable/commit/618b396) on `dev`
