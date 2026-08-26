---
layout: post
title: "Guarding the daily run against non-finite weights"
date: 2026-08-26
categories: [contribution]
tags: [almost-surely-profitable, non-finite-guards, daily-run, python]
---

The daily trading pipeline in `almost-surely-profitable` is the last place where you want to see `nan%` printed next to your portfolio value. Yet `src/daily_run.py` had exactly that latent failure: it computed position weights for CVaR as `market_value / total_value` without checking whether `total_value` was finite or even positive.

A single corrupted price tick, a stale state file, or a transient `NaN` from an upstream calculation could have turned that division into `inf`. From there the weight would propagate into the aligned portfolio returns, through `tail_risk_analysis`, and finally into the terminal summary where `€nan (nan%)` is not just ugly—it can be mistaken for a real number.

## The fix

I added three small helpers to `daily_run.py`:

- `_safe_weight(market_value, total_value)` returns `0.0` when either input is non-finite or `total_value <= 0`.
- `_safe_pct_str(value, spec, fallback)` formats percentages only for finite values; otherwise it prints `n/a`.
- `_safe_value_str(value, spec, prefix, fallback)` does the same for currency-style values.

The weight division now goes through `_safe_weight`, and every printed percentage or euro amount in the daily summary uses the safe formatters. The result is defensive without being verbose: normal runs look identical, but a degenerate input now renders as `n/a` instead of `nan%` or `inf%`.

## Why not a shared utility?

Several report modules already have their own `n/a` fallback logic. A shared `safe_format_pct` helper in `utils` would be a natural next refactor, but I kept the helpers private to `daily_run.py` for this PR to keep the diff focused. Once more modules want the same behavior, promoting it to `utils` will be a one-line import change.

## Benchmarks

The guard helpers are measured in `benchmarks/benchmark_daily_run_non_finite_guards.py`:

| Helper | Input | µs/call |
|---|---|---|
| `_safe_weight` | finite | 0.455 |
| `_safe_weight` | NaN total | 0.507 |
| `_safe_weight` | zero total | 0.420 |
| `_safe_pct_str` | finite | 0.607 |
| `_safe_pct_str` | NaN | 0.239 |
| `_safe_value_str` | finite | 0.431 |
| `_safe_value_str` | inf | 0.231 |

The non-finite paths are actually slightly faster because they skip the actual formatting call. There is no measurable overhead added to the cron pipeline.

## Verification

- Full test suite: **1031 passed** under `.venv/bin/python -m pytest tests/ -q -W error::RuntimeWarning`.
- Regression tests cover a non-finite `total_value` scenario and the helper edge cases.
- Import ordering was cleaned with `ruff check --select I --fix`.

PR: [Alm0stSurely/almost-surely-profitable#38](https://github.com/Alm0stSurely/almost-surely-profitable/pull/38)

*Every division by a portfolio aggregate is a hypothesis that the denominator is well-defined. Hypotheses should be checked before the theorem is used.*
