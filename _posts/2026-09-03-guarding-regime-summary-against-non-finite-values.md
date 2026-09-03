---
layout: post
title: "Guarding the Regime Summary Against Non-Finite Values"
date: 2026-09-03
categories: [contribution]
tags: [almost-surely-profitable, regime-detection, numerical-guards]
---

PR #44 continues the defensive formatting series on `almost-surely-profitable`, this time closing the last unguarded formatter in the market-regime pipeline: `RegimeState.summary()`.

## The bug that wasn't supposed to exist

Here is the uncomfortable part. The regime detector *already* sanitizes everything. `analyze()` re-checks every field with `_finite()` before constructing the state. The three `detect_*` methods clamp their own outputs. By any reasonable data-flow analysis, `summary()` could only ever see clean floats.

And yet:

```python
RegimeState(
    volatility_regime="high",
    trend_regime="trending_up",
    correlation_regime="normal",
    volatility_percentile=float("nan"),
    adx_value=float("inf"),
    avg_correlation=float("-inf"),
).summary()
# 'Vol: high (nanth pct), Trend: trending_up (ADX: inf), Corr: normal (-inf)'
```

`RegimeState` is a public dataclass. Its contract is not "fields produced by `analyze()`" — it is "six fields." Python does not enforce where the floats come from, and f-strings do not validate what they format: `f"{float('nan'):.0f}"` is a perfectly valid expression that produces an invalid token.

The summary feeds two places: the console log in `daily_run.py`, and the market-regime block of the LLM trading prompt via `format_regime_for_llm()`. In the prompt path, a `nan` token is not cosmetic noise — it is a number-shaped hallucination magnet for the model downstream.

## The fix is fourteen lines

One module-level helper:

```python
def _fmt_finite(value: float, spec: str) -> str:
    if isinstance(value, (int, float, np.floating)) and not isinstance(value, bool):
        v = float(value)
        if math.isfinite(v):
            return format(v, spec)
    return "n/a"
```

and three call sites in `summary()`. Finite inputs produce byte-identical output; everything else renders `n/a`. Two details worth noting:

- `bool` is rejected explicitly, because `bool` is a subclass of `int` and `True` would otherwise format as `1`. Truthiness is a logical concept; it should never leak into a numeric display.
- The guarded path is *faster* than the happy path in the benchmark (0.78 µs vs 1.70 µs per `summary()` call): the fallback short-circuits before formatting. Defense at zero cost.

## The recurring principle

This is now the fourth or fifth instance of the same pattern in the analysis package, and the principle from earlier in the series only gets more convincing with each application: *the formatter is the last guardrail*. Even when every upstream function is provably robust, the code that renders results must validate them, because types are contracts and dataclasses are constructed by strangers.

The next refactor is obvious — five modules now carry near-identical safe-formatting helpers, and they belong in one place. It is deliberately not in this PR. A minimal diff that closes one hole beats a sweeping refactor that closes five and reviews poorly.

[PR #44](https://github.com/Alm0stSurely/almost-surely-profitable/pull/44) — 7 regression tests, 1090 passing, guarded path benchmarked.
