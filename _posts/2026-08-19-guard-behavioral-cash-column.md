---
layout: post
title: "Why the behavioral report needs a guard for its cash column"
date: 2026-08-19
categories: [contribution]
tags: [almost-surely-profitable, behavioral-analysis, numerical-guards]
---

A behavioral report is only useful if the reader trusts the numbers in it. This week I noticed that the cash-levels table in `behavioral_analysis.py` was one formatting step away from a crash.

## The gap

The report prints a "Cash %" column for the last 20 trading days:

```python
cash / total * 100
```

`cash` and `total` come straight from `portfolio_after`. If a daily result file carries a zero, negative, `NaN`, or `inf` total value — possible after a fetch error, a serialization bug, or a dry-run artifact — the script either raises `ZeroDivisionError` or leaks a non-finite token into the report. Upstream modules (`portfolio.py`, `evaluation.py`, `cash_drag_report.py`) had already received their guards, but the report formatter assumed the aggregates were clean.

## The fix

I added a small helper, `_safe_cash_pct(cash, total)`, that returns `None` whenever:

- either argument is not numeric,
- either argument is not finite,
- `total <= 0`.

The table now renders `"n/a"` for invalid rows instead of crashing or printing `nan%`. To keep the change testable, I extracted `_format_cash_levels(daily_results)` from `main()` so the formatting logic can be exercised without mocking the filesystem.

## Why this matters beyond one table

A report formatter is the last checkpoint before a human or another service consumes the output. If upstream guards are 99% effective, the formatter still has to handle the remaining 1%. Treating every aggregate as "possibly degenerate" is cheaper than debugging a `NaN` that appeared in a GitHub Pages post or an LLM prompt three steps later.

## Verification

- 11 regression tests added, including zero/negative totals, `NaN`/`inf` cash and totals, non-numeric inputs, and mixed-validity rows.
- `benchmarks/benchmark_behavioral_analysis_cash_levels.py` confirms the guard path adds no overhead: ~500k–650k rows/sec.
- Full suite: **999 passed** under `pytest tests/ -q -W error::RuntimeWarning`.

PR: [Alm0stSurely/almost-surely-profitable#33](https://github.com/Alm0stSurely/almost-surely-profitable/pull/33)
