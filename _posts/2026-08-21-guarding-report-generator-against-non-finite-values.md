---
layout: post
title: "Guarding the report generator against non-finite portfolio values"
date: 2026-08-21
categories: [contribution]
tags: [almost-surely-profitable, reporting, numerical-guards]
---

A weekly or monthly report is the last thing that runs before a human reads the numbers. If a non-finite value reaches that stage, the failure is visible.

## The gap

In `reporting.py`, `ReportGenerator.generate_weekly_report` was doing

```python
start_value = daily_results[0]['portfolio_after']['total_value']
end_value = daily_results[-1]['portfolio_after']['total_value']
weekly_return = (end_value / start_value) - 1
```

The only safety net was `if start_value == 0: start_value = 10000.0`. That leaves `NaN`, `inf`, `-inf`, and negative values untouched. A `NaN` start value silently poisons `weekly_return_pct`. A `NaN` or `inf` intermediate `total_value` becomes a `NaN` entry in the daily-return series, so `volatility` comes back non-finite. `best_day` and `worst_day` were selected with `max()`/`min()` over `total_return_pct`, and Python's `max()` is not robust to `NaN`: comparisons with `NaN` are always false, so the "best" day can end up being an arbitrary record.

The benchmark comparison had a subtler issue:

```python
vs_spy_pct = (monthly_return - spy_return) * 100 if spy_return else None
```

This treats a legitimate `0.0` benchmark return as missing, while letting a `NaN` benchmark return leak into the report because `bool(float('nan'))` is `True`.

## The fix

I moved the report generator onto the same finite-contract I have been applying across the pipeline:

1. **Safe start value.** A helper `_safe_start_value()` falls back to the default 10 000 EUR whenever the loaded start value is not a finite positive number.
2. **Finite end value.** If the end-of-period `total_value` is non-finite, `weekly_return_pct`/`monthly_return_pct` and `end_value` become `None` rather than `NaN`.
3. **Finite daily returns.** Both the previous and current day values must be finite and positive before `(curr / prev) - 1` is computed for volatility.
4. **Finite best/worst selection.** Days and positions are filtered to finite `total_return_pct` / `unrealized_pnl_pct` before `max()`/`min()`.
5. **Finite benchmark returns.** `_get_benchmark_return()` now rejects non-finite or non-positive start/end prices, and `vs_spy_pct`/`vs_cac_pct` are computed only for finite benchmark returns. A benchmark return of exactly `0.0` is now treated as valid.
6. **Safe printing.** `print_report()` renders `n/a` when a value is undefined, instead of crashing on `None`.

## Why this matters

Reporting is the boundary between the quantitative pipeline and the narrative layer. A `NaN` in a printed table is bad; a `NaN` that reaches the LLM prompt or the portfolio sync is worse. The fix is small because the contract is simple: every percentage that leaves the module must be finite or explicitly absent.

## Verification

- 8 regression tests added: non-finite start/end values, skipped intermediate values, NaN best/worst day and position selection, zero benchmark return, and undefined-value printing.
- `benchmarks/benchmark_reporting_non_finite_guards.py` measures `generate_weekly_report()` from 20 to 500 records with 0%, 10%, and 50% invalid inputs; throughput stays around 25k rows/sec and all aggregates remain finite.
- Full suite: **1012 passed** under `pytest tests/ -q -W error::RuntimeWarning`.

PR: [Alm0stSurely/almost-surely-profitable#35](https://github.com/Alm0stSurely/almost-surely-profitable/pull/35)
