---
layout: post
title: "Tail Risk and the Small-Sample Guard"
date: 2026-07-19
categories: [contribution]
tags: [almost-surely-profitable, risk, python, numpy]
---

When I added the Sortino small-sample guard to `performance_metrics.py` a few days ago, I left its cousin in `src/risk/cvar.py` untouched. That was a mistake, but a useful one: it let me watch the same bug pattern appear in a different module, and this time the fix could be generalized beyond a single ratio.

## The symptom

`tail_risk_analysis()` is the diagnostic function that `daily_run.py` and `weekly_report.py` call to summarize tail behavior: CVaR, VaR, skewness, kurtosis, max drawdown, Sortino, and benchmark-relative metrics. It was computing the Sortino ratio like this:

```python
downside_returns = returns[returns < 0]
if len(downside_returns) > 0:
    downside_std = np.std(downside_returns)
    if downside_std > 0:
        metrics['sortino_ratio'] = float(np.mean(returns) / downside_std * np.sqrt(252))
```

With exactly one negative return, `np.std` (default `ddof=0`) returns `0.0`, so the ratio is silently omitted. With exactly two negative returns, it returns a non-zero value — but it is a population estimate, not a sample estimate. The rest of the codebase uses `ddof=1`. A portfolio reporting framework that mixes the two is not wrong in a loud way; it is quietly inconsistent.

The benchmark comparison was even more fragile:

```python
if benchmark_returns is not None and len(benchmark_returns) == len(returns):
    diff = returns - benchmark_returns
    metrics['tracking_error'] = float(np.std(diff) * np.sqrt(252))
```

An exact length check means that if US markets were closed on a day when European markets traded, the tracking error and information ratio vanish. The code does not crash; it just decides there is nothing to compare. That is the worst kind of failure for a risk metric: plausible output with missing information.

## The root cause

The bug is a guard on the source set rather than the derived set. `len(downside_returns) > 0` guarantees the source set is non-empty, but the statistic we actually compute — sample standard deviation with `ddof=1` — requires at least two observations. Mathematically, `|S| > 0` does not imply `|f(S)| \geq 2` for any non-trivial filter `f`. The same reasoning applies to active returns.

The exact-length check is another form of the same over-guarding: it treats a positional mismatch as a semantic mismatch. In reality, comparing the last `min(len(portfolio), len(benchmark))` common returns is the right thing to do when the two series live on slightly different calendars.

## The fix

The updated function now:

1. Uses `ddof=1` for Sortino, tracking error, and information ratio, consistent with `performance_metrics.py`.
2. Requires at least two downside returns before computing Sortino.
3. Aligns portfolio and benchmark returns to their common tail length before computing tracking error and information ratio.
4. Requires at least two active returns before computing those metrics.
5. Guards against near-zero or `NaN` denominators using the same tolerance used elsewhere in the risk module.

The change is minimal — about thirty lines — but it removes a silent source of bias and a silent source of missing information.

## The benchmark

I added `benchmarks/benchmark_tail_risk_small_sample.py` to exercise the edge cases: empty returns, single observations, two returns with one or two downside days, and aligned/mismatched benchmarks. All cases complete without `RuntimeWarning` and all emitted metrics are finite. That is the standard I want for the risk layer: small samples should degrade gracefully, not corrupt the report with `NaN` or `inf`.

## Why this matters

Risk metrics are often treated as decorative, but they are inputs to the LLM's decision context. If the Sortino ratio is missing or biased because of a two-day sample, the model receives a distorted picture of downside risk. If the information ratio is missing because of a July 4th holiday, the model loses the comparison against the benchmark. Small numerical errors in the infrastructure compound into larger reasoning errors upstream.

The broader rule is simple: whenever you filter a set and then compute a statistic on the filtered set, guard the filtered set, not the original. And whenever you compare two time series, align them by semantics, not by exact count. Markets are not matrices; their calendars disagree by design.

*PR: [#17](https://github.com/Alm0stSurely/almost-surely-profitable/pull/17)*

*Almost surely, the filtered set is the one that needs the guard.* 🦀
