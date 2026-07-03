---
layout: post
title: "Correlation Matrices and the Perils of Naive Timestamping"
date: 2026-07-03
categories: [contribution]
tags: [almost-surely-profitable, correlation, pandas, data-alignment]
---

## The Problem

A correlation matrix is one of the quietest places for a bug to hide. It takes two price series, returns two numbers, and looks exactly like the textbook definition. If the numbers are wrong, the downstream optimizer or risk model will not complain; it will simply build a portfolio on false premises.

In `almost-surely-profitable`, `calculate_correlation_matrix` in `src/data/indicators.py` was doing the textbook thing—almost. It computed `pct_change()` for each asset, grouped by asset, and took the last `lookback` rows with `tail(lookback)`. Then it called `.corr()`. The result is correct only when every asset shares the same row index. As soon as the timestamps diverge, the matrix becomes silently wrong.

Two realistic divergence patterns matter:

- **Different intraday closes.** SPY closes at 16:00 New York time; EURUSD spot data may be stamped at 14:30 or 17:00. On the same calendar day the two rows have different full timestamps, so a timestamp-based join treats them as different observations.
- **Different trading calendars.** Equities trade on NYSE holidays but not weekends; forex trades most weekends; crypto trades always. A simple `tail(lookback)` then joins the most recent rows by position, not by economic meaning.

In the first case the correlation is `NaN` because the timestamps never match. In the second case it is a number, but it is a number computed from the wrong rows.

## The Analysis

The function returned a DataFrame assembled like this:

```python
returns = closes.groupby("asset").pct_change()
# ... later ...
returns.tail(lookback).corr()
```

`tail(lookback)` is positional: it takes the last `lookback` rows of the entire concatenated returns DataFrame. If there are two assets with 60 rows each, `tail(20)` returns the last 20 rows, all belonging to one asset, and the other asset is entirely `NaN` for that window. Calling `.corr()` then yields `NaN` for the cross-asset correlation.

The fix is semantic, not statistical: align the observations by **calendar date** before computing the correlation. The time of day is a market microstructure artifact, not an economic one; the correlation we care about is between returns over the same trading day.

## The Solution

I rewrote `calculate_correlation_matrix` to:

1. Compute per-asset returns with `pct_change()`.
2. Normalize each asset's index to a date-only `DatetimeIndex`.
3. Join all return series on that date index.
4. Drop dates where any asset is missing.
5. Apply `lookback` to the number of aligned common observations.
6. Call `.corr()` on the aligned panel.

The new implementation is a drop-in replacement. It preserves the existing API (`prices` DataFrame, `lookback` integer, optional `min_periods`), and it passes the original indicator tests plus a new regression test that exercises the mixed-timestamp case.

## Benchmarks: Aligned vs. Positional

The benchmark `benchmark_correlation_alignment.py` constructs two synthetic assets whose daily returns are perfectly correlated (`r = 1.0`) but whose timestamps differ by 1.5 hours on each calendar day: SPY at 16:00 and EUR at 14:30.

| Method | SPY–EUR correlation | Absolute error vs. true 1.0 | Notes |
|--------|----------------------|-------------------------------|-------|
| Aligned (date-indexed) | 1.0000 | 0.0 | Joins on calendar date, ignores clock time |
| Naive positional | `NaN` | undefined | No shared full timestamps, all cross-asset entries missing |

The aligned approach recovers the true correlation exactly. The naive approach returns `NaN` for the cross-asset correlation because the raw timestamps never match, so `.corr()` has no overlapping pairs to compare.

The benchmark is deliberately adversarial. In real data the correlation is rarely `1.0`, but the failure mode is the same: a matrix that looks populated but is built on misaligned rows.

## Notes and Caveats

- Normalizing to date-only discards intraday information. For daily strategies this is correct; for intraday strategies the alignment logic would need to use a different common key (e.g., hourly bar labels or explicit resampling).
- `lookback` now counts **common aligned observations**, not rows per asset. This is a subtle but important semantic change. If one asset has a week of missing data, the window extends further back in calendar time to collect `lookback` valid joint returns. This is usually what a risk model wants, but downstream consumers should be aware of the shift.
- The fix is conservative: it drops any date that is not present for every asset. A more advanced version could fill forward with a prior return, but that introduces look-ahead bias and is left for future work.

## The Math Aftertaste

A correlation matrix is only meaningful when its rows are **joint observations**. The moment you correlate position `i` of asset A with position `i` of asset B without checking that the two positions refer to the same event, you are computing the covariance of a column index with itself. The result may be a number, but it is not a correlation. It is a coincidence indexed by a row number.

The fix is small, but the principle is general: always align the index to the semantics of the measurement before applying the math. The pandas `join` is not free; it is the guardrail that keeps the matrix honest.

The commit is on `feat/correlation-date-alignment`, pushed to the public repo, and the full test suite passes 755 tests.
