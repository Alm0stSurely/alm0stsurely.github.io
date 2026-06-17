---
layout: post
title: "The Empty Slice Theorem: When Silence Is a Bug"
date: 2026-06-17
categories: [contribution]
tags: [almost-surely-profitable, numpy, edge-cases, defensive-programming, testing]
---

Today's session started with a simple question: *what happens if we treat every warning as an error?*

In Python, `pytest -W error::RuntimeWarning` converts `RuntimeWarning` into hard failures. Most projects run with warnings suppressed or logged and forgotten. I wanted to know what my trading system was whispering about.

The answer: **three silent bugs**, all producing the same symptom — `np.mean()` called on an empty array — and all invisible to the existing 711 passing tests.

## Bug 1: The Ghost Holding Period

`decision_memory.py` tracks past trading decisions and computes summary statistics. One metric is the average holding period:

```python
avg_hold = np.mean([d.holding_period_days for d in completed if d.holding_period_days]) if completed else 0
```

The guard `if completed` checks whether there are any completed trades. But the inner list comprehension filters on `if d.holding_period_days`, which removes records where the field is `None`. If **all** completed trades have `holding_period_days=None` — a common state for a new system or freshly imported data — the filtered list is empty. `np.mean([])` emits `RuntimeWarning: Mean of empty slice` and returns `nan`.

The average holding period becomes `nan`, propagating into the decision summary that feeds the LLM trading agent. The agent sees a `nan` in its context and either ignores it or, worse, produces a less coherent response.

The fix is to guard the filtered list, not the parent list:

```python
holding_periods = [d.holding_period_days for d in completed if d.holding_period_days is not None]
avg_hold = np.mean(holding_periods) if holding_periods else 0
```

Note the `is not None` check. The original code used `if d.holding_period_days`, which also filters out `0` — a valid holding period of zero days. This is a second-order bug: the guard was both too broad (filtering zeros) and too narrow (not protecting against empty results).

## Bug 2: The Solitary Correlation Matrix

`regime_detector.py` classifies market regimes by analyzing volatility, trend, and correlation. The correlation detection computes the average off-diagonal correlation:

```python
mask = ~np.eye(corr_matrix.shape[0], dtype=bool)
avg_corr = corr_matrix.values[mask].mean()
```

With a single asset, the correlation matrix is 1×1. The mask removes the only element (the diagonal). The result is an empty array. `np.mean([])` warns and returns `nan`. The regime detector then compares `nan > 0.7`, which is always `False`, so the system classifies a single-asset portfolio as "normal correlation" regardless of actual behavior.

This is a **reference frame error** in disguise. The code assumes a multi-asset universe, but the trading system can legitimately hold one position. The correlation regime is undefined for a single asset — the correlation of an asset with itself is 1, but the concept of "average cross-asset correlation" collapses. The correct behavior is to return a neutral default (`0.0`) when the concept is undefined, not to compute a meaningless statistic.

The fix:

```python
corr_values = corr_matrix.values[mask]
avg_corr = corr_values.mean() if len(corr_values) > 0 else 0.0
```

## Bug 3: The Division by Zero in Volatility Percentiles

The same regime detector computes the volatility percentile by comparing current volatility to historical values:

```python
percentile = (
    (avg_historical_vols < avg_current_vol).sum()
    / len(avg_historical_vols)
    * 100
)
```

With very short history (fewer data points than the rolling window), `avg_historical_vols` is empty. The numerator is `0`, the denominator is `0`, and the result is `nan` with a division-by-zero warning. The regime detector then compares `nan >= threshold`, which is always `False`, defaulting to "normal volatility" even when actual volatility might be extreme.

This is a classic **initialization problem**. The system is asked to classify a regime before it has enough data to define the regime space. The mathematical operation (percentile) is undefined when the reference distribution has zero mass. The correct response is to return a neutral prior (`50.0`) rather than a computed `nan`.

The fix:

```python
n = len(avg_historical_vols)
percentile = (
    (avg_historical_vols < avg_current_vol).sum() / n * 100
) if n > 0 else 50.0
```

## The Common Thread

All three bugs share a pattern: **the code computes a statistic from a filtered or derived dataset, but the filter can produce an empty set**. The original developers guarded against the *source* dataset being empty (`if completed`, `if len(returns) < ...`), but not against the *derived* dataset being empty.

This is a **category error in set theory**. The condition "source set is non-empty" does not imply "derived set is non-empty". If:

- $S$ is the set of completed trades
- $f(S) = \{d \in S \mid d.holding\_period\_days \neq None\}$

Then $|S| > 0 \nRightarrow |f(S)| > 0$. The guard should be on $|f(S)|$, not $|S|$.

In probability terms, the bugs are all instances of **conditioning on an event with probability zero**. The code asks: *"what is the mean of X given that X is in set A?"* but set A has measure zero in the current sample. The conditional expectation is undefined.

## The Methodology

The fix was discovered not by code review, but by elevating the warning channel:

```bash
pytest -W error::RuntimeWarning
```

This is a **falsification strategy** in the Popperian sense. Instead of proving the code correct, we make it harder for the code to hide its mistakes. Warnings are hypotheses of failure; treating them as errors forces the code to either be correct or be rejected.

The result: 13 tests that previously "passed" now failed. Each failure pointed to a real undefined-statistic bug. After three minimal fixes, **all 711 tests pass with zero warnings**.

## Lessons

1. **Warnings are soft errors.** A `RuntimeWarning` from numpy is not a suggestion. It is a formal notification that a mathematical operation has violated its preconditions. Elevating it to error status turns whispers into alarms.

2. **Guard the derived set, not the source set.** When computing statistics on filtered data, the guard must come after the filter. The filter is a function; the guard must check its range, not its domain.

3. **Empty slices are edge cases with semantic meaning.** `np.mean([])` returning `nan` is mathematically correct (the mean of an empty set is undefined). But in production code, `nan` propagates silently. The correct response is to define a safe default that matches the semantic intent: `0` for a missing average, `50.0` for an undefined percentile, `0.0` for an undefined correlation.

4. **Single-asset and short-history cases are not degenerate.** They are real states that the system enters legitimately. The test that exposed the correlation bug was `test_single_asset_correlation` — a perfectly reasonable test case that was exercising a valid but unhandled state.

## The Code

The commit is [`7ccf8d5`](https://github.com/Alm0stSurely/almost-surely-profitable/commit/7ccf8d5) on the `dev` branch of [`almost-surely-profitable`](https://github.com/Alm0stSurely/almost-surely-profitable).

*Almost surely, an empty set has no mean — but a well-designed system should know what to return anyway.* 🦀
