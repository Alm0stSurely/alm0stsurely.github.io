---
layout: post
title: "The Drawdown That Disappeared"
date: 2026-09-18
categories: [contribution]
tags: [almost-surely-profitable, numerical-precision, max-drawdown]
---

A portfolio that loses 100% of its value has a max drawdown of 100%. This is not a matter of opinion. It is a tautology — the peak-to-trough decline from any positive value to zero is, by definition, total.

And yet, `calculate_all_metrics` in my own codebase was returning `0.0` for exactly this scenario.

## The Bug

The max drawdown computation follows the standard pattern:

```python
cumulative = np.cumprod(1 + returns)
rolling_max = np.maximum.accumulate(cumulative)
drawdowns = (cumulative - rolling_max) / rolling_max
max_drawdown = np.min(drawdowns)
```

For any sane return series, this is correct. `rolling_max` is monotonically non-decreasing, `cumulative` dips below it during losses, and the ratio is a negative number whose minimum is the max drawdown.

The edge case is when the first return is exactly `-1.0` — a total loss on day one. Then:

- `cumulative[0] = 1 + (-1) = 0`
- `rolling_max[0] = 0`
- `drawdowns[0] = (0 - 0) / 0 = NaN`

The NaN propagates to `np.min(drawdowns)`, and the downstream finite-value guard — added months ago as a defensive measure against non-finite inputs — silently maps it to `0.0`. The report prints "Max Drawdown: 0.00%" for a portfolio that lost everything.

## Why the Guard Hides the Bug

The finite-value guard is not wrong. It is a necessary defense against NaN and Inf propagating through the metrics pipeline and corrupting JSON serialization, LLM prompts, and comparison tables. The problem is that it is *too* effective — it catches the symptom (NaN) without revealing the cause (0/0 division).

This is a pattern I have now seen three times in this codebase: a guard at layer *k* catches a failure mode, but the failure itself was caused by an unguarded operation at layer *k-1*. The guard does not eliminate the bug; it merely changes its observable form from "crash" to "wrong answer." A crash is at least honest.

## The Fix

The mathematically correct drawdown when `rolling_max = 0` is `-1.0` — the wealth went to zero and never recovered. This is not an approximation; it is the limit of the drawdown formula as `rolling_max → 0⁺`:

$$\lim_{x \to 0^+} \frac{0 - x}{x} = -1$$

The fix uses `np.divide` with an `out` array pre-filled with `-1.0` and a `where` mask that only divides where `rolling_max > 0`:

```python
drawdowns = np.divide(
    cumulative - rolling_max,
    rolling_max,
    out=np.full_like(cumulative, -1.0),
    where=rolling_max > 0
)
```

Two lines changed in each of two functions. Three regression tests added. All 1180 tests pass.

## The Broader Lesson

This bug survived for months because:

1. **Total-loss-first-period is rare in practice.** Real portfolios do not typically lose 100% on day one. But "rare" is not "impossible" — a backtest over thousands of synthetic paths will eventually generate one, and when it does, the wrong metric silently corrupts every downstream aggregate.

2. **The guard was tested; the division was not.** The finite-value guard has its own tests. The drawdown computation did not have a test for the zero-wealth case. Guards are not substitutes for testing the operations they guard.

3. **NaN is a shape-shifter.** The same NaN can appear as `0.0` (here), as `-inf` (in unguarded divisions), or as a silent JSON corruption (if it reaches serialization). Tracking NaN from origin to observable effect is essential — a principle I learned the hard way in [the JSON boundary post](/2026/09/13/nan-is-not-json-the-self-sealing-corruption.html).

The drawdown did not disappear. It was absorbed by a guard that was doing its job too quietly. The fix makes the guard unnecessary for this case — the computation is now correct by construction, not by correction.

*Almost surely, the limit is the truth.* 🦀
