---
layout: post
title: "Testing Decision Quality: Measuring the Oracle's Prophetic Accuracy"
date: 2026-05-15
categories: [testing, math]
tags: [almost-surely-profitable, decision-analysis, backtesting, sharpe]
---

A trading system without post-hoc decision analysis is a black box with a slot machine handle. You pull it, money comes out (or doesn't), and you have no idea whether the mechanism inside is a sophisticated model or a coin flip with extra steps.

Today's work: comprehensive tests for the `decision_analyzer.py` module — the component that evaluates whether the LLM's trading decisions were actually any good. Forty-two tests, all green, covering a module that was previously operating on faith.

## What Decision Analysis Measures

The analyzer answers a simple question with a complicated answer: *was the LLM right?*

Simple because the definition of "right" is binary: after a buy, did the price go up? After a sell, did the price go down? Complicated because:

1. **Forward returns are path-dependent.** A 5-day forward window captures short-term momentum but misses multi-week trends. A 1-day window captures overnight reaction but is dominated by noise.
2. **The counterfactual is unobservable.** When the LLM sells, we measure the price drop we "avoided" — but we don't observe the world where we held. The "saved return" is a statistical construct, not a ground truth.
3. **Success rates are Bernoulli trials with unknown p.** A 60% win rate over 20 trades is not statistically distinguishable from random (p ≈ 0.26 under H₀: p = 0.5). You need ~100 trades for a 55% rate to be significant at α = 0.05.

## The Test Architecture

### Loading Decisions from JSON

The analyzer reads `results/daily/*.json` — the actual trading decisions produced by the pipeline. The tests verify:

- **Missing directory**: graceful degradation with a warning, not a crash.
- **Invalid JSON**: skipped with a warning, other files continue loading.
- **Missing keys**: files without `decision` or `executed_trades` are silently skipped.
- **Day limiting**: `load_decisions(days=5)` only loads the 5 most recent files, respecting chronological order.

This is defensive programming for a filesystem interface. In production, files get corrupted, disks fill up, and cron jobs write half-finished JSON. The analyzer must not crash on messy data.

### Forward Return Calculation

The core computation:

```python
forward_return = (exit_price - entry_price) / entry_price
```

This is trivial arithmetic. The complexity is in the data plumbing:

- **Timezone handling**: yfinance returns timezone-aware DatetimeIndex. The decision date is a string. We normalize both to timezone-naive before comparison, avoiding the `TypeError: Cannot compare tz-naive and tz-aware timestamps` that has bitten every pandas user at least once.
- **Missing data**: if the entry date is after all available data, the mask is empty and we return 0.0. Not ideal, but safe.
- **Exceptions**: any network error, parse error, or data anomaly returns 0.0 silently. The analyzer is a diagnostic tool, not a trading engine. It must not crash the pipeline.

### Outcome Classification

The tests verify the correctness of the success predicate:

| Action | Forward Return | Success? | Interpretation |
|--------|---------------|----------|----------------|
| Buy | +5% | ✓ | Bought before a rise |
| Buy | -2% | ✗ | Bought before a drop |
| Sell | -3% | ✓ | Sold before a drop (avoided loss) |
| Sell | +4% | ✗ | Sold before a rise (missed gains) |

The sell case is the subtle one. The forward return is negated in the Sharpe calculation because selling before a drop is good, even though the raw return is negative. The metric `avg_forward_return_sell` is defined as `-np.mean(sell_returns)`, so a positive value means the sells were, on average, timed before drops.

### Behavioral Patterns

Beyond raw accuracy, the analyzer tracks behavioral metrics:

- **Overconfidence**: >4 trades per day scores 0.4/1.0. The LLM has shown a tendency to overtrade when volatility spikes — this metric quantifies the problem.
- **Diversification**: `unique_assets_traded / 10`, capped at 1.0. A one-asset portfolio scores 0.1, which is accurate for risk concentration.
- **Loss aversion**: `min(2 * sell_ratio, 1.0)`. All buys = 0.0 (never cuts losses). All sells = 1.0 (always exits). The scaling by 2 means a 50/50 buy/sell split scores 1.0 — interpreted as "appropriate loss aversion in a balanced market."

The loss aversion formula is a heuristic, not a formal model. It assumes that in a declining market, sells are good. This is directionally true but ignores the information content: was the sell at -5% or -50%? A stop-loss at -5% is discipline; a panic sell at -50% is capitulation. The current metric can't distinguish them. Future work.

### The Sharpe of Decisions

The pseudo-Sharpe ratio of decision returns is:

```python
sharpe = mean(decision_returns) / std(decision_returns)
```

Where `decision_returns` inverts sell returns (so selling before a drop contributes positively). This is not a true Sharpe ratio — there's no risk-free rate, and the returns are not log-returns — but it's a useful relative metric.

The test verifies edge cases:
- Single decision: Sharpe = 0.0 (undefined variance → safe default).
- Zero volatility: Sharpe = 0.0 (division by zero guard).
- Mixed outcomes: positive mean with non-zero variance yields positive Sharpe.

## What the Tests Caught

Writing tests for `decision_analyzer.py` revealed no new bugs — the module was already functional. But they did reveal *design assumptions* that were never explicitly documented:

1. **The 0.0 return convention.** When data is missing, forward return is 0.0, which counts as a failure. This biases the win rate downward for illiquid assets or recent listings. Is this conservative or pessimistic? It depends on your prior.
2. **The sell return inversion.** The negation of sell returns in the Sharpe calculation is mathematically sound but semantically subtle. A test is the only documentation that guarantees this behavior persists through refactoring.
3. **Timezone normalization.** The code handles timezone-aware and timezone-naive indices. Without tests, this logic would be the first casualty of a "cleanup" refactor.

## The Test Count

We now have **253 tests** across the codebase:
- 42 for decision analysis (new today)
- 50 for regime detection
- 27 for evaluation
- 23 for reporting
- 18 for backtest cooldowns
- 14 for position cooldown
- 12 for trading agent
- 11 for portfolio
- 10 for performance metrics
- 9 for backtest engine
- 8 for fetch market data
- 7 for indicators
- 6 for monitor
- 4 for CVaR
- 4 for decision memory (placeholder)

The decision analyzer closes another gap in the diagnostic pipeline. We can now measure not just *what* the LLM decided, but whether those decisions were any good — with statistical rigor and explicit edge-case handling.

## Next Steps

The remaining untested modules are:
1. `decision_memory.py` — long-term decision tracking and persistence
2. `churn_analysis.py` — position turnover and fee analysis
3. `daily_run.py` integration — the orchestration layer that ties everything together

As the sample size of trades grows, the decision analyzer will move from descriptive (what happened) to inferential (is the LLM better than random?). With 253 tests guarding the pipeline, we can trust the measurements almost surely.

*253 tests. The law of large numbers starts at N=1, but statistical significance requires a bit more patience.*
