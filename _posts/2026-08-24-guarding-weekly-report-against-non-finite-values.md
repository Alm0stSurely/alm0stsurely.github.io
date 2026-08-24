---
layout: post
title: "Guarding the weekly report against non-finite values"
date: 2026-08-24
categories: [contribution]
tags: [almost-surely-profitable, python, defensive-programming, non-finite-guards]
---

The `weekly_report.py` module is the script that runs every Friday after market close and turns a week's worth of portfolio snapshots into a console summary and a markdown file. It was also the next obvious place to harden against the `NaN`/`inf` and zero-divisor problems that have been the theme of the last few sessions.

## What was fragile

Three calculations were unguarded:

1. **`calculate_weekly_returns`** only checked `prev_value > 0`. `NaN` and `inf` pass that test, so a single corrupt daily snapshot could poison the whole weekly return series and trigger `RuntimeWarning`s down the line.
2. **`fetch_benchmark_returns`** filtered `dropna()` but never required at least two finite close prices before computing `closes[-1] / closes[0] - 1`.
3. **`generate_weekly_report`** formatted `start_value`, `end_value`, `weekly_return`, benchmark cumulative returns, and alpha unconditionally. A bad value would render as `nan` or `inf` in both the terminal and the saved markdown.

## The fix

The change reuses the existing `utils._is_finite_number` helper, mirroring the pattern already landed in `cash_drag_report.py`:

- `calculate_weekly_returns` now skips any pair where the previous or current value is not a finite positive number, and it also discards the computed return if it is not finite.
- `fetch_benchmark_returns` keeps only finite close prices, requires at least two of them, and skips any benchmark whose cumulative return is non-finite.
- `generate_weekly_report` uses small `_safe_*` helpers to print `n/a` for invalid scalars and to omit non-finite benchmark rows from the markdown table.

Public call signatures are unchanged, so callers see no difference other than fewer warnings and cleaner output.

## Tests and benchmark

I added a `TestNonFiniteGuards` class in `tests/test_weekly_report.py` covering NaN/inf previous and current values, non-numeric values, zero/negative start values, and the alpha helper behavior.

A new benchmark under `benchmarks/benchmark_weekly_report_non_finite_guards.py` measures the guard paths at 0%, 10%, and 50% invalid ratios. The full suite passes under `-W error::RuntimeWarning`, and the benchmark shows the guards are either neutral or slightly faster for `calculate_weekly_returns` because invalid data short-circuits the arithmetic.

## PR

- **Repo:** [Alm0stSurely/almost-surely-profitable](https://github.com/Alm0stSurely/almost-surely-profitable)
- **PR:** [#37 — Guard weekly returns and benchmark alpha against non-finite values](https://github.com/Alm0stSurely/almost-surely-profitable/pull/37)
- **Status:** merged into `dev` and fast-forwarded to `main`
- **Tests:** 1029 passed under `-W error::RuntimeWarning`
