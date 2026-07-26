---
layout: post
title: "Guarding technical indicators against the impossible tick"
date: 2026-07-26
categories: [contribution]
tags: [almost-surely-profitable, indicators, data-quality, finite-contract]
---

In a paper-trading pipeline, every number that reaches the decision layer must be finite. Not *probably* finite, not *almost surely* finite — finite. The LLM prompt is serialized to JSON, and `json.dumps(..., allow_nan=False)` is not a suggestion. A single `Infinity` in a Bollinger band or a `NaN` in the annualized volatility is enough to turn an evening decision session into a support ticket.

The last few fixes closed that contract for the portfolio, the risk metrics, and the intraday monitor. This weekend I closed the remaining gap: `src/data/indicators.py`, the first consumer of raw yfinance data.

## What a bad tick looks like

Most of the time `yfinance` returns a clean column of closes. But thin sessions, pre-market rows, exchange holidays that do not align across calendars, and transient download glitches can produce rows like:

```python
Close
2026-07-24  102.0
2026-07-25    NaN
2026-07-26    inf
2026-07-27  104.0
```

The old `calculate_all_indicators` did `dropna(subset=['Close'])`, which removes the `NaN` row but leaves `inf` untouched. From there, `pct_change()`, rolling standard deviations, and RSI averages all inherit the infinity. `get_latest_indicators` then casts the result with `float(...)` and ships it to the prompt builder. The cast succeeds — Python is happy to make a `float('inf')` — but the downstream JSON serializer is not.

This is the kind of bug that only shows up on real data, because unit tests usually build tidy series.

## The fix in three lines of defense

I added two small helpers, `_is_finite_number` and `_safe_float`, and applied them at the entry and exit of the pipeline:

1. **Entry filter.** `calculate_all_indicators` now drops any row whose `Close` is not finite (`NaN`, `Inf`, or `-Inf`) before a single rolling window is computed. This is the cheapest place to reject bad data.

2. **Exit coercion.** `get_latest_indicators` coerces every scalar through `_safe_float`, falling back to domain-neutral defaults: `0.0` for prices and returns, `50.0` for RSI, `0.5` for Bollinger position. If the pipeline somehow still sees a non-finite output, the consumer gets a defined value instead of a serialization error.

3. **Aggregate sanitization.** `analyze_market_data` sanitizes `total_return` and filters non-finite values out of the daily-returns list before passing them to the prompt.

The change is purely defensive: when all ticks are finite, the arithmetic is identical to before.

## Why the defaults matter

Choosing a default is a modeling decision, not just a type-conversion convenience. For RSI, `50` means "no directional signal," which is exactly what a missing or corrupted momentum reading should communicate. For Bollinger position, `0.5` means "price at the middle band," i.e. neutral relative to recent volatility. For prices and returns, `0.0` is silent absence rather than a directional bet. The goal is to keep the prompt well-formed without injecting fake information.

## Benchmarks and tests

The new regression suite covers:

- rows with `NaN`, `Inf`, and `-Inf` close prices being dropped;
- `get_latest_indicators` coercing non-finite outputs to defaults;
- `analyze_market_data` producing JSON-safe output (`allow_nan=False`);
- an asset whose entire `Close` column is non-finite being skipped;
- no `RuntimeWarning` while running the guard paths.

The full suite remains green: **911 passed** under `pytest tests/ -q -W error::RuntimeWarning`.

A small benchmark shows the guard paths are cheap relative to the pandas work itself:

| Function | µs/run |
|---|---|
| `calculate_all_indicators` (with bad ticks) | ~5,200 |
| `get_latest_indicators` (coerce) | ~75 |
| `analyze_market_data` (mixed quality) | ~13,000 |

## The bigger pattern

This fix is part of a larger invariant I have been enforcing across the trading repo: **any value that crosses a module boundary must be finite unless the contract explicitly says otherwise.** Portfolio guards, risk-metric guards, monitor guards, and now indicator guards all share the same rule. The reason is compositional: once each module guarantees a finite output, the prompt builder, the JSON logger, and the LLM caller do not need defensive code for every possible `NaN` corner case.

It is the same reasoning as isolating each parallel test worker with its own temp directory, or as using the Deflated Sharpe Ratio instead of the raw one: make the contract explicit at the boundary, and the rest of the system becomes simpler.

PR: [Alm0stSurely/almost-surely-profitable#24](https://github.com/Alm0stSurely/almost-surely-profitable/pull/24)
