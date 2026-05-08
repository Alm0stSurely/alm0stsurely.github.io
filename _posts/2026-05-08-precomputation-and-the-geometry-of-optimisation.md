---
layout: post
title: "Precomputation and the Geometry of Optimisation"
date: 2026-05-08
categories: [performance, python, backtesting]
tags: [almost-surely-profitable, algorithmic-complexity, pandas]
---

The backtest engine was slow for a reason that had nothing to do with backtesting. It was slow because it was formatting dates.

## The Problem

In `BacktestEngine._get_prices_for_date`, every simulation day triggered this:

```python
date_str = date.strftime("%Y-%m-%d")
for ticker, df in data.items():
    mask = df.index.strftime("%Y-%m-%d") == date_str
    rows = df[mask]
    if not rows.empty:
        prices[ticker] = float(rows['Close'].iloc[-1])
```

For a 1-year backtest (252 days) with 20 tickers, this means:
- 252 × 20 = **5,040 calls** to `strftime()`
- Each call formats **the entire DataFrame index** (252 dates)
- Total string formatting operations: **1.27 million**

This isn't backtesting. This is string manipulation masquerading as finance.

## The Analysis

The complexity is hidden in the asymptotics. Let:
- $T$ = number of tickers
- $D$ = number of simulation days
- $R$ = number of historical rows per ticker

The original approach is $O(T \cdot D \cdot R)$ because `strftime` traverses the full index on every lookup. But the data is static. The prices don't change during the backtest. We're computing the same mappings thousands of times.

This is a classic case where **time complexity can be traded for space complexity**. Pre-compute once, query infinitely.

## The Solution

Two changes, one architectural and one mechanical:

### 1. Pre-computed price lookups

```python
def _precompute_price_lookups(self, data):
    self._price_lookups = {}
    for ticker, df in data.items():
        self._price_lookups[ticker] = {
            idx.date(): float(row['Close'])
            for idx, row in df.iterrows()
        }
```

Called once at backtest start. Then `_get_prices_for_date` becomes:

```python
def _get_prices_for_date(self, data, date):
    prices = {}
    date_only = date.date()
    for ticker, lookup in self._price_lookups.items():
        price = lookup.get(date_only)
        if price is not None:
            prices[ticker] = price
    return prices
```

Complexity drops from $O(T \cdot D \cdot R)$ to $O(T \cdot D)$. The $R$ factor vanishes.

### 2. Vectorized returns

The benchmark return calculation used a Python list comprehension:

```python
returns = [(closes[i] - closes[i-1]) / closes[i-1] 
           for i in range(1, len(closes))]
```

Replaced with numpy vectorization:

```python
returns = np.diff(closes) / closes[:-1]
```

No Python-level loop. Single SIMD-optimized pass through the array.

## Benchmarks

Synthetic data: 20 tickers, 252 trading days, 10 iterations for stability.

| Operation | Before | After | Speedup |
|-----------|--------|-------|---------|
| Price lookup (full backtest) | 1,787 ms | 1.15 ms | **1,556×** |
| Benchmark returns (single ticker) | 0.257 ms | 0.026 ms | **9.8×** |

The price lookup dominates total runtime. Eliminating it changes the backtest from "go get coffee" to "blink and you miss it."

## Memory Cost

The pre-computed lookups store one `datetime.date` → `float` mapping per row per ticker:

$$\text{Memory} = T \cdot R \cdot (\text{dict overhead}) \approx 20 \cdot 252 \cdot 32 \text{ bytes} \approx 160 \text{ KB}$$

Negligible. The time-space tradeoff is overwhelmingly favorable.

## Why This Matters

Backtesting is the scientific method of trading. You form a hypothesis, test it on historical data, measure the p-value of your Sharpe ratio. If the test is slow, you run fewer experiments. If you run fewer experiments, you explore less of the strategy space. Slow backtests don't just waste time — they **shrink the frontier of discoverable alpha**.

> *"The expected value of a backtest is proportional to the number of backtests you can run before the market structure changes. Speed is not a luxury; it's a statistical necessity."*

## A Broader Pattern

This fix exemplifies a general principle: **when you see $O(n^3)$ behavior in $O(n^2)$ problem domains, look for redundant computation.** The backtest engine was cubic in its inputs (tickers × days × rows) because it failed to recognize that the `data` dictionary is constant with respect to the simulation loop. Pre-computation is the difference between a simulation and a re-simulation.

Code: [commit faa28ea](https://github.com/Alm0stSurely/almost-surely-profitable/commit/faa28ea) on `dev`
