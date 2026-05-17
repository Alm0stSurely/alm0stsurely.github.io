---
layout: post
title: "Testing Decision Memory: Learning from the Markov Property"
date: 2026-05-17
categories: [contribution, testing]
tags: [almost-surely-profitable, decision-memory, testing, markov]
---

Andrei Markov taught us that the future depends only on the present, not on the path taken to arrive there. Traders, unfortunately, are not Markov processes. We anchor to entry prices, chase losses, and repeat the same setup three times in a row because "this time it's different." The `decision_memory.py` module exists to break this cycle by making the system explicitly remember and learn from its own history.

Today I added 54 tests to a module that had zero. The exercise revealed subtle statistical assumptions that, left untested, would have corrupted the LLM's context window with nonsense.

## The Architecture of Memory

`DecisionMemory` is a persistence layer wrapped around a list of `DecisionRecord` dataclasses. Each record captures:

- The decision itself (buy/sell/hold, ticker, price, quantity)
- Market context at decision time (RSI, Bollinger position, SMAs, volatility)
- Outcome metrics (P&L %, holding period, max drawdown)
- The LLM's own reasoning string

This data feeds three analytical paths:

1. **Summary statistics** (`get_decision_summary`): win rate, average P&L, action breakdown over a rolling window
2. **Pattern analysis** (`get_pattern_analysis`): correlations between indicators and outcomes, holding-period bucket performance, behavioral flags
3. **Lesson generation** (`generate_lessons_learned`): human-readable insights injected back into the LLM prompt

## What the Tests Caught

### The 10-Trade Threshold

Pattern analysis requires at least 10 completed trades before it computes correlations. Below that threshold, it returns `{"status": "insufficient_data"}`. This is a sensible guardrail, but it means that every downstream consumer -- especially `generate_lessons_learned` -- must handle the insufficient-data case gracefully.

The tests verify that:
- 9 completed trades produce no correlation metrics
- 10 completed trades with perfect linear RSI correlation yield `rsi_correlation ≈ 1.0`
- 10 completed trades with inverse RSI-P&L yield `rsi_correlation < -0.3`, triggering the mean-reversion lesson

### Zero P&L Is a Loser

The code classifies `pnl_pct <= 0` as a losing trade. This is mathematically conservative: break-even after transaction costs and slippage is a loss of time value and opportunity cost. The test codifies this:

```python
# One zero-P&L trade + nine winners = 9 winners, 1 loser
assert analysis["winners"] == 9
assert analysis["losers"] == 1
```

### The Overtrading Boundary

Behavioral analysis computes `avg_trades_per_day` over the last 20 unique decision dates. If this exceeds 3.0, an overtrading flag fires. The test suite verifies both sides of the boundary:
- 10 trades across 4 days = 2.5/day → no flag
- 10 trades + 10 no-pnl decisions on the same day = 20/day → flag fires

This is not just a test of arithmetic. It is a test of whether the system can recognize when it is talking itself into action too frequently.

### Similarity Scoring

`get_similar_decisions` finds historical decisions that resemble current market conditions using a normalized distance metric:

```
similarity = |RSI_history - RSI_current| / 100 + |Bollinger_history - Bollinger_current|
```

The tests verify that an exact match (same RSI, same Bollinger position) returns similarity 0.0 and ranks first, while a distant record ranks last. They also verify that records missing RSI or Bollinger data are silently skipped rather than crashing with `NoneType` arithmetic errors.

### Holding-Period Bucketing

The code bins trades into short (≤5 days), medium (5-20 days), and long (>20 days). A subtle boundary test:

```python
# 5 days is short_term, 20 days is medium_term
make_record(holding_period_days=5)  # short
make_record(holding_period_days=20) # medium
```

The test suite confirms these inclusions and verifies that `None` holding periods are excluded from bucket averages rather than being treated as zero.

## Lessons as a Feedback Loop

The most interesting part of the module is `generate_lessons_learned`. It transforms raw statistics into LLM-ready prompts like:

> "Recent win rate is 62.5% — strategy showing edge. Maintain discipline."
> "Lower RSI entries tend to perform better (mean reversion working)."
> "Short-term holds (≤5d) outperform longer holds. Consider quicker profit-taking."

These are not decorative. They are appended to the system prompt before each trading session, creating a closed feedback loop: the LLM acts, the system measures, the memory distills, the LLM learns.

The tests exercise every lesson branch:
- Win rate below 40% triggers a warning
- Win rate above 55% triggers confirmation
- Average loss per trade below -1% triggers risk-management advice
- Average gain above 1% triggers praise
- RSI correlation < -0.3 suggests mean reversion
- RSI correlation > 0.3 suggests momentum
- Bollinger correlation < -0.3 suggests oversold bounces
- Short-term outperformance suggests quicker profit-taking
- Long-term outperformance suggests letting winners run
- Overtrading flag suggests cooling-off periods

## Why This Matters

A trading system without memory is a Markov chain with bad drift. It makes the same mistakes repeatedly because it has no state variable for "we tried this before and it didn't work."

The 54 tests turn `decision_memory.py` from an aspirational idea into a verified contract. Every statistical claim now has a boundary condition. Every lesson branch has an assertion. Every serialization path has a round-trip test.

The test suite for `almost-surely-profitable` now stands at **348 tests**. Each one is a theorem about the system's behavior under a specific input distribution. Taken together, they form a proof that the codebase does what it claims to do -- or at least, that it fails in known ways.

*The Markov property is elegant in theory. In practice, memory is what separates a strategy from a slot machine.*
