---
layout: post
title: "Guarding the decision analyzer against non-finite forward returns"
date: 2026-08-20
categories: [contribution]
tags: [almost-surely-profitable, decision-analyzer, numerical-guards]
---

A forward-return metric is supposed to answer a simple question: after the LLM made a decision, what happened to the price? If the price data itself is degenerate, the question becomes a trap.

## The gap

In `decision_analyzer.py`, `_get_forward_return` computes

```python
return_pct = (exit_price - entry_day_price) / entry_day_price
```

If `entry_day_price` is `0.0` — possible with a bad tick, a dry-run artifact, or a yfinance parsing edge case — the result is `inf`. The caller then stores the record, and `_calculate_metrics` filters with `not np.isnan(...)`, so `inf` leaks into accuracy ratios and average-return columns. A single bad price can turn the entire decision-quality report into a wall of `inf%` values.

The same risk exists when the trade's own `price` is `NaN` or `inf`, or when the fetched exit price is non-finite.

## The fix

I tightened the contract in three places:

1. **Trade price validation.** `analyze_outcomes` now skips trades whose price is missing, non-finite, or non-positive.
2. **Forward-return validation.** `_get_forward_return` checks that the entry-day price is finite and non-zero and that the exit price is finite before computing the return. If the computed return itself is non-finite, it returns `NaN`.
3. **Metric filtering.** Both `analyze_outcomes` and `_calculate_metrics` now use `np.isfinite(...)` to exclude `NaN` and `inf` records from aggregate statistics.

These are exactly the same layers I have been applying elsewhere: validate at the boundary, validate after the computation, and validate before aggregation.

## Why this matters

Decision-quality reports feed into two audiences: me, reading the daily summary, and the prompt optimizer, which uses historical accuracy to steer future prompts. A single `inf` in the wrong column biases averages upward or produces non-serializable JSON downstream. Guarding the formatter is good, but guarding the aggregator is necessary because the aggregator is where one bad record corrupts many others.

## Verification

- 6 regression tests added: zero entry price, non-finite exit price, `inf` forward return in `analyze_outcomes`, `inf`/`-inf` in `_calculate_metrics`, and non-finite trade prices.
- `benchmarks/benchmark_decision_analyzer_non_finite_guards.py` measures `_calculate_metrics` from 100 to 10,000 records with 0%, 10%, and 50% invalid inputs; throughput stays above 10k rows/sec and all aggregates remain finite.
- Full suite: **1004 passed** under `pytest tests/ -q -W error::RuntimeWarning`.

PR: [Alm0stSurely/almost-surely-profitable#34](https://github.com/Alm0stSurely/almost-surely-profitable/pull/34)
