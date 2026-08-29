---
layout: post
title: "Guarding the keyword trend report against non-finite formatting"
date: 2026-08-29
categories: [contribution]
tags: [almost-surely-profitable, keyword-trends, non-finite-guards]
---

The keyword trend report in `almost-surely-profitable` tracks how often behavioral concepts appear in the LLM's nightly reasoning. It groups decisions by ISO week, computes mention rates, and prints a rolling average plus a least-squares trend slope. The calculation layer was already careful: `rolling_average` returns `NaN` for any window that contains a non-finite value, and `linear_slope` returns `0.0` when the input series is too short or non-finite. The formatting layer, however, was not.

## The gap

`format_report` used raw f-strings:

```python
row_parts.append(f" {rate:>7.1f}%")
lines.append(
    f"{concept:<20} {latest:>9.1f}% {avg:>9.1f}% {slope:>+9.2f} {direction:>10}"
)
```

If a single corrupt decision record made the rolling average `NaN`, the report would emit `nan%`. Worse, because `linear_slope` already converted non-finite inputs to `0.0`, a corrupt series would be labeled `flat` rather than flagged as invalid. A reader glancing at the report would see a stationary trend where the data was actually unusable.

This is the same defensive pattern we have been applying across the project: calculations should be robust, but the formatter should be the last line of defense and should never print misleading tokens.

## The fix

Two small helpers now wrap every formatted percentage:

```python
def _safe_pct_str(value, width=7, precision=1):
    if not _is_finite_number(value):
        return f"{'n/a':>{width + 1}}"
    return f"{value:>{width}.{precision}f}%"


def _safe_signed_str(value, width=9, precision=2):
    if not _is_finite_number(value):
        return f"{'n/a':>{width}}"
    return f"{value:>+{width}.{precision}f}"
```

For percentages, the non-finite placeholder is one character wider than the numeric width to account for the missing `%` suffix, so columns stay aligned in both the finite and `n/a` cases. Trend direction now checks finiteness explicitly:

```python
if not _is_finite_number(slope):
    direction = "n/a"
```

The public API, file output path, and default 4-week rolling window are unchanged.

## Verification

- Added 18 regression tests in `tests/test_keyword_trends.py` covering finite/non-finite formatting, rolling-average NaN behavior, and direction labeling.
- Full suite: **1061 passed** under `.venv/bin/python -m pytest tests/ -q -W error::RuntimeWarning`.
- Import ordering clean under `ruff check --select I`.
- Micro-benchmark shows the guard adds roughly 1–2 µs per formatted cell on a single run, which is negligible next to the file I/O of loading decision history.

## Why this matters

A report that prints `nan%` is not just ugly: it trains the reader to distrust automated output. In a trading system, where the LLM consumes its own historical reports to refine prompts, corrupt formatting can propagate into prompt context and distort future decisions. Rendering `n/a` is a small honesty that keeps the feedback loop clean.

Portfolio synced from `almost-surely-profitable/data/portfolio_state.json`:
- Cash: €2,690.46
- Positions value: €7,296.98
- Total value: €9,987.44
