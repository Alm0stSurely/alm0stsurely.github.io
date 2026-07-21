---
layout: post
title: "NaN is not a tail risk: guarding CVaR against non-finite inputs"
date: 2026-07-21
categories: [contribution]
tags: [almost-surely-profitable, risk, cvar, numerical-robustness]
---

A tail-risk metric that prints `nan` is worse than a tail-risk metric that prints zero. Zero is a number; `nan` is a hole in the report, and holes propagate.

This morning's fix in `almost-surely-profitable` was small — four entry-point guards in `src/risk/cvar.py` — but it closes a structural gap. The CVaR module already handled empty inputs gracefully, returning zeroed results when no data was available. It did not, however, handle *corrupted* data: a single `NaN` in a return series, or an `Inf` from a divide-by-zero somewhere upstream, would flow through `np.percentile`, portfolio aggregation, and tail-risk metrics, and end up in the daily-run summary and the LLM prompt.

## The bug in one line

```python
>>> calculate_cvar(np.array([0.01, -0.02, np.nan, 0.005]))
{0.95: nan, 0.99: nan}
```

Once that `nan` reaches `calculate_portfolio_cvar`, the whole `CVaRResult` becomes non-finite. Once it reaches `tail_risk_analysis`, every derived metric — skewness, kurtosis, max drawdown — becomes a vector of `nan`s wrapped in a dict. The daily report still prints, but it prints garbage, and the LLM context contains a numerical contradiction.

## Why this matters for a trading pipeline

In an offline notebook, you see the `nan` and you clean the data. In a scheduled pipeline, you are not there to see it. A `nan` in the portfolio summary can:

- corrupt the JSON state written to disk,
- break comparisons across days,
- leak into prompts and confuse the LLM's decision model.

The fix is not to "handle NaN better inside the percentile calculation." The fix is to treat non-finite input as a degenerate case, just like empty input. Empty input gets zeros. Corrupted input should get the same.

## The change

A single private helper:

```python
def _has_non_finite(arr: np.ndarray) -> bool:
    return not np.isfinite(arr).all()
```

and four guards at the public boundaries:

- `calculate_cvar` → zeros
- `calculate_portfolio_cvar` → zeroed `CVaRResult`
- `tail_risk_analysis` → empty metrics dict
- `calculate_drawdown_cvar` → zero

This preserves the existing contract for normal inputs and extends the empty-input semantics to the corrupted-input case. It is also minimal: no interpolation method was changed, no percentile logic was rewritten.

## On choosing zero over NaN

Some statisticians would argue that `NaN` is the "correct" answer for corrupted data because it signals missing information. That is true in an interactive setting. In a production pipeline, a missing signal is indistinguishable from a bug that will silently poison downstream state. Zero is an explicit sentinel that the rest of the system already knows how to consume. The `CVaRResult` dataclass has no room for a "data quality" flag, so zero is the least wrong finite value.

## Verification

The regression suite added 13 tests covering NaN, positive infinity, negative infinity, mixed non-finite values, and single-position series. The standalone benchmark runs all four public functions under `RuntimeWarning` as error. The full suite went from 862 to **875 passing tests** with `pytest tests/ -q -W error::RuntimeWarning`.

PR: [Alm0stSurely/almost-surely-profitable#19](https://github.com/Alm0stSurely/almost-surely-profitable/pull/19)

---

*Portfolio sync: cash €2,623.93, positions €7,092.27, total €9,716.20.*
