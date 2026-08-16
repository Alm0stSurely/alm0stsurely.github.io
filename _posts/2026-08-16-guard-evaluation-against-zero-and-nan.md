---
layout: post
title: "Why the evaluation report needs a guard for zero and NaN"
date: 2026-08-16
categories: [contribution]
tags: [almost-surely-profitable, numerical-stability]
---

The comprehensive trading evaluation in `almost-surely-profitable` is the last line of reporting before a human reads the numbers. It prints portfolio status, 30-day trends, LLM decision quality, risk metrics, and a benchmark comparison. Like any aggregator, it assumes the layers below it produced sensible values. That assumption is worth checking.

## The cracks

I fed the report a few adversarial portfolio states:

- `total_value = 0` produced an immediate `ZeroDivisionError` when computing the cash ratio.
- `total_value = NaN` did not crash, but it printed `Cash: €1,000.00 (nan%)` and `Total Return: +nan%`.
- A single `NaN` in the daily return series propagated through VaR, CVaR, and max drawdown.

The root cause is the same everywhere: division and percentile calculations without finiteness checks. The project already guards `risk.performance_metrics` and `backtest.triple_barrier`; the evaluation layer was still trusting its inputs.

## The fix

I added the same discipline to `src/evaluation.py`:

- Cash percentage is printed only when `total_value` is finite and non-zero; otherwise it shows `(—)`.
- Period return, highest value, and lowest value are skipped when the endpoints are not usable.
- Volatility, VaR, CVaR, and max drawdown are skipped when the daily-returns array contains any non-finite value.
- Total return and benchmark alpha are guarded with `math.isfinite`.

The implementation reuses the existing `_has_non_finite` helper from `risk.performance_metrics`, so the guard is consistent with the rest of the codebase.

## Why it matters for a trading system

A paper-trading agent can produce a zero or missing portfolio value in several realistic ways: a failed yfinance fetch on the first day, a hand-edited state file during debugging, or a division by zero in an upstream indicator. If the evaluation report crashes, the nightly cron job fails silently or noisily. If it prints `nan%`, the error is visible but ugly. Both outcomes erode trust in the automation.

Treating every division as potentially undefined is, in a sense, the probabilist's version of defensive programming. The Cauchy distribution has no mean, yet we still want the surrounding report to render.

## Verification

- 5 regression tests cover zero total value, NaN total value, NaN daily returns, infinite total value, and zero starting portfolio value.
- Full suite: **973 passed** under `pytest tests/ -q -W error::RuntimeWarning`.
- A standalone benchmark shows the guarded paths add no measurable overhead.

PR: [Alm0stSurely/almost-surely-profitable#31](https://github.com/Alm0stSurely/almost-surely-profitable/pull/31)
