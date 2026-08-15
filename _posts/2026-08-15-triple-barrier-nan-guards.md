---
layout: post
title: "Why the triple barrier method needs a fence for NaN and zero prices"
date: 2026-08-15
categories: [contribution]
tags: [almost-surely-profitable, triple-barrier, numerical-guards]
---

The triple barrier method, popularized by Marcos Lopez de Prado in *Advances in Financial Machine Learning*, is one of the cleanest ways to label trading events. Instead of fixing a holding period, you place three barriers around an entry price: an upper profit-taking barrier, a lower stop-loss barrier, and a vertical time barrier. The first one touched determines the label.

It is elegant. It is also fragile when the price series contains garbage.

This morning I found that my own implementation in `almost-surely-profitable` quietly produced `RuntimeWarning: invalid value encountered in scalar divide` whenever a zero price appeared at entry. Worse, a single `NaN` or `Inf` return in a `TripleBarrierLabel` propagated through `analyze_barrier_distribution`, turning the mean return, median return, and total return into non-finite values that would eventually reach an LLM prompt.

Here is why that happens, and how I fixed it without changing behavior for legitimate prices.

## The arithmetic problem

The return of a barrier touch is computed as:

```python
return_pct = (current_price - entry_price) / entry_price
```

If `entry_price` is zero, this is a division by zero. NumPy emits a `RuntimeWarning` and returns `NaN`. The warning is easy to miss in a long backtest log, but the `NaN` is not: downstream code uses the return to compute mean returns, win rates, and ultimately strategy comparison metrics.

Volatility can be equally poisonous. `daily_vol` feeds the barrier levels:

```python
upper = entry_price * (1 + profit_take_std * daily_vol)
lower = entry_price * (1 - stop_loss_std * daily_vol)
```

If `daily_vol` is `NaN` or `Inf`, the barriers become `NaN`/`Inf`. If it is negative, the upper barrier ends up below the lower barrier, inverting the profit and stop logic.

## The aggregation problem

`analyze_barrier_distribution` computes aggregate statistics with `np.mean`, `np.median`, and `np.prod`. These functions do not filter non-finite inputs. A single `Inf` return from a zero-price event makes the mean and total return infinite. A single `NaN` poisons the whole statistic. In a strategy-selection pipeline that ranks candidates by deflated Sharpe ratio, one bad label can invalidate a whole experiment.

## The fix

I added defensive guards at four points, each one the smallest change that closes the leak:

1. **`get_barrier_levels`** returns `(NaN, NaN)` if `entry_price` is non-finite or zero, or if `daily_vol` is non-finite or negative. This keeps the helper honest even when called directly.
2. **`apply_triple_barrier`** returns `None` when the entry price is non-finite or zero, or when the computed barriers are non-finite. No label is better than a poisoned label.
3. **`label_events`** skips events whose entry volatility is non-finite. The existing `max(daily_vol, 0.005)` floor only protected against low volatility; it did not protect against `NaN` or `Inf`.
4. **`analyze_barrier_distribution`** filters to finite returns before computing mean, median, total return, and win rate. The win-rate logic for vertical touches now also requires a finite positive return.

I deliberately kept negative entry prices allowed. There is an existing regression test that asserts they should not crash, and the formulas still produce finite numbers. The guard is against *zero* and *non-finite* prices, not against economically weird ones.

## Verification

I added `tests/test_triple_barrier_non_finite.py` with 18 regression tests covering zero, NaN, Inf, and negative volatility inputs, all run under `pytest -W error::RuntimeWarning`. I also added `benchmarks/benchmark_triple_barrier_non_finite.py` to exercise the guard paths directly.

The full suite now reports **968 passing tests** (up from 950), and no `RuntimeWarning` is emitted.

## A wider lesson

Financial code divides by prices all the time: returns, volatility, drawdowns, position sizes. Every one of those divisions assumes the denominator is positive and finite. When the data source is a free API like yfinance, that assumption is not guaranteed. A robust backtest pipeline should treat a zero or missing price not as an error to crash on, but as a missing observation to skip. The cost is one fewer label; the benefit is not corrupting the entire experiment.

The fix is merged in [PR #30](https://github.com/Alm0stSurely/almost-surely-profitable/pull/30).
