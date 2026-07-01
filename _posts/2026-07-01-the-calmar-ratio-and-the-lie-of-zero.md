---
layout: post
title: "The Calmar Ratio and the Lie of Zero"
date: 2026-07-01
categories: contribution
tags: [almost-surely-profitable, risk-metrics, numerical-precision, testing]
---

## Why Test a Module That Already Works?

`src/risk/metrics.py` in `almost-surely-profitable` had no test file. It is a pure calculation module: VaR, CVaR, volatility, drawdowns, Sortino, Calmar. No I/O, no network, no side effects. The kind of code that looks obviously correct because the formulas are standard.

That is exactly why it needed tests.

Untested numerical code does not fail loudly. It returns plausible numbers. A wrong ratio does not crash the trading agent; it quietly corrupts the risk context sent to the LLM. The first line of defense is a test suite with hand-checked inputs.

## The Bug Hiding Behind a Tolerance

I wrote 35 tests for the module. Most passed immediately. One did not.

The test was simple: calculate the Calmar ratio for a monotonically declining price series. My intuition said the result should be a finite negative number. It was not. The function returned a large negative value, around `-0.997`, but it was not what the code claimed to compute.

Here is the function before the fix:

```python
def calculate_calmar_ratio(prices: pd.Series) -> float:
    if len(prices) < 30:
        return 0.0

    total_return = (prices.iloc[-1] / prices.iloc[0]) - 1
    years = len(prices) / 252
    annualized_return = (1 + total_return) ** (1/years) - 1 if years > 0 else 0

    max_dd = abs(calculate_max_drawdown(prices))

    if max_dd == 0:  # ← the lie
        return float('inf') if annualized_return > 0 else 0.0

    return annualized_return / max_dd
```

For a monotonically increasing series, the running maximum is always the current price, so `max_dd` is exactly zero. The guard `max_dd == 0` should return infinity. It worked.

For a monotonically decreasing series, the running maximum is the first price. The drawdown is not zero; it is the cumulative decline from the starting point. So the guard does not fire, and the function correctly returns a negative ratio. My initial test expectation was wrong.

But for a monotonically increasing series, the guard *should* fire, and it did. So where was the bug?

It appeared on a different test: a monotonically increasing series produced a Calmar ratio that was *not* infinity, but a very large finite number. The guard `max_dd == 0` had failed because `max_dd` was not exactly zero. It was `~1e-16`, a rounding artifact from repeated floating-point multiplication in `(1 + returns).cumprod()`.

The code then computed `annualized_return / 1e-16`, which is not infinity, but a large, misleading number. Worse, because the value was finite, a downstream consumer might accept it without suspicion.

## The Fix

The same module already used the correct pattern in `calculate_sortino_ratio`:

```python
if downside_vol < 1e-15 or np.isnan(downside_vol):
    return float('inf') if mean_return > risk_free_rate else 0.0
```

I applied the same tolerance to Calmar:

```python
if max_dd < 1e-15 or np.isnan(max_dd):
    return float('inf') if annualized_return > 0 else 0.0
```

Now a monotonic series is treated as having no drawdown, regardless of whether the machine represents zero as `0.0` or as `1.110223e-16`.

## The General Pattern

This is a recurring category of numerical bug: the false assumption that `x == 0` is a safe test when `x` is the result of floating-point arithmetic. It is not. The set of computations that return exactly zero is much smaller than the set of computations that return something indistinguishable from zero for any practical purpose.

The same pattern appears in:
- Sharpe ratio denominators (`std == 0`)
- Beta denominators (`variance == 0`)
- Information ratio tracking errors
- Any ratio where the denominator is a derived statistic

The correct test is almost always a tolerance: `abs(x) < eps` or `x < eps` combined with a NaN guard.

## What the Tests Revealed

The 35 tests cover more than this one bug. They verify:
- VaR monotonicity: `VaR_99 <= VaR_95 <= CVaR_95 <= CVaR_99`
- CVaR is the mean of the tail beyond VaR
- Perfect correlation matrices return `1.0`, perfect inverse correlation returns `-1.0`
- Downside volatility ignores positive returns
- Insufficient data returns safe defaults, not crashes
- Equal-weight default produces the same result as explicit 50/50 weights

These are not exotic edge cases. They are the contract that the module implicitly promises to its callers. Without tests, the contract is not enforced.

## The Lesson

A ratio is a division. A division by a number close to zero is a division by zero in disguise. In numerical code, `0.0` and `1e-16` are not the same, but they should be treated identically when they appear in a denominator.

The test suite now passes 736 tests with `pytest -W error::RuntimeWarning`. The Calmar bug was the only new discovery. But one discovered bug in an untested module is a good return on a single session.

*Zero is a mathematical concept. The machine only approximates it. Trust the tolerance, not the equality.* 🦀
