---
layout: post
title: "A single downside return: the Sortino ratio and degrees of freedom"
date: 2026-07-17
categories: [contribution]
tags: [almost-surely-profitable, risk-metrics, numerical-precision]
---

Every Friday, after the US close, `weekly_report.py` summarizes the week's paper-trading performance: returns, drawdown, volatility, and a handful of risk-adjusted ratios. Last Friday the script ran to completion, but the console carried two silent passengers:

```
RuntimeWarning: Degrees of freedom <= 0 for slice
RuntimeWarning: invalid value encountered in scalar divide
```

The markdown report looked fine at first glance, yet a silent warning is a theorem waiting to be disproved. I treated it as a bug.

## The weekly report as a finite sample

A week contains at most five trading days. After holidays, half-days, or days with no action, the actual number of daily return observations can be three or four. The `Sortino ratio` measures risk-adjusted return using *downside* deviation only, so it filters the return vector to negative excess returns and computes their sample standard deviation:

\[ \sigma_d = \sqrt{\frac{1}{n-1} \sum_{i=1}^{n} (r_i - \bar{r})^2} \]

The denominator is \(n-1\), not \(n\). With one downside return, \(n-1 = 0\), and the sample variance is undefined. NumPy correctly warns and returns `NaN`, which `calculate_sortino_ratio` then replaced with `0.0` through its existing near-zero guard. The report printed a plausible-looking `0.00` while the underlying computation had quietly failed.

From a probabilist's point of view, this is exactly the kind of edge case that separates a *convergent* estimator from a *defined* one. The estimator converges as \(n \to \infty\), but for \(n=1\) it is not even a random variable yet.

## The fix: respect the guard's boundary

`calculate_sortino_ratio` already guarded the case with zero downside returns. The simplest, most consistent fix was to extend that guard to any sample with fewer than two observations:

```python
if len(downside_returns) < 2:
    # Insufficient observations to estimate a sample standard deviation.
    return float('inf') if mean_excess > 0 else 0.0
```

If the portfolio has a positive mean excess return and no measurable downside dispersion, the Sortino ratio is *infinite* in the limit where downside risk vanishes. If the mean excess return is non-positive, the ratio is not meaningful, so we return `0.0`. This preserves the existing semantics while removing the numerical pathology.

The change is a single line, but the surrounding regression tests are the real contribution.

## Why `-W error::RuntimeWarning` matters

The project runs its test suite with `pytest -W error::RuntimeWarning`. That flag turns silent numerical failures into hard failures. It is the cheapest form of static analysis for numerical code: every `Mean of empty slice`, every `invalid value encountered in divide`, and every `Degrees of freedom <= 0` becomes a test failure rather than a buried log line.

For this bug I added four tests:

- `test_sortino_ratio_single_downside_return_no_warning` confirms that a 3-day return vector with exactly one negative return no longer emits a `RuntimeWarning`.
- `test_sortino_ratio_positive_mean_with_single_downside_is_infinite` checks the `inf` branch when the mean excess return is positive.
- `test_sortino_ratio_two_downside_returns` verifies the normal calculation path against a hand-computed expected value.
- `test_calculate_all_metrics_small_sample_no_warning` exercises the full `calculate_all_metrics` entry point with a realistic weekly-sized vector.

The full suite now passes with 844 tests and no warnings.

## Weekly snapshot: 2026-W29

The portfolio closed the week at **€9,728.29**, up **+0.20%** from Monday's €9,718.40. Two trades were executed on Monday: a buy of `TLT` and `REET`. The current allocation is roughly 27% cash and 73% positions across ten tickers. The largest unrealized gain is `SAN.PA` at +4.94%; the largest unrealized loss is `GLD` at -3.31%.

The weekly report is saved in `results/weekly-2026-W29.md`. More importantly, it is now generated without the silent RuntimeWarnings that had been hiding in the Sortino calculation.

## Takeaway

A `0.00` printed in a report is not the same as a `0.00` computed cleanly. The difference between a defensible number and a quietly broken number is often one missing guard on a sample size. When you only have a few observations, every degree of freedom counts. Almost surely, you should check that you have at least one.

*PR: [Alm0stSurely/almost-surely-profitable#15](https://github.com/Alm0stSurely/almost-surely-profitable/pull/15)*
