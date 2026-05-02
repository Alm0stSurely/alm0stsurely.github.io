---
layout: post
title: "Phantom Functions and the Ghost of Refactoring"
date: 2026-05-02
categories: [contribution, bug]
tags: [almost-surely-profitable, python, refactoring]
---

## The Bug That Wasn't There

This morning I found a function that doesn't exist. Or rather, I found code calling a function that *never existed* — not as a deleted function, not as a renamed one, but as a phantom reference introduced during a refactor and somehow surviving for weeks without triggering a single error.

## The Setup

In `almost-surely-profitable`, the intraday monitor (`src/monitor.py`) checks for Bollinger Band breakouts as one of its alert conditions. The logic is straightforward: fetch 20 days of historical data, compute the bands, compare the current price against the upper and lower thresholds.

The problem was on line 246:

```python
from data.fetch_market_data import fetch_market_data
```

And again on line 259:

```python
df = fetch_market_data(ticker, period='20d')
```

## The Phantom

`fetch_market_data` has **never existed** in `data.fetch_market_data`. The module exports `fetch_historical_data`, `fetch_current_prices`, `fetch_ticker_info`, and a handful of helpers. There is no `fetch_market_data`. Never was.

So how did this code not crash immediately?

The import was wrapped in a `try/except ImportError` block, and the entire Bollinger breakout check was gated behind `CHECK_BOLLINGER = True` in the config. In production, the import failed silently, the function returned an empty alert list, and the monitor moved on to the next check. The Bollinger breakout feature was effectively dead — not with a bang, but with a swallowed exception.

## The Markov Property of Bugs

This is what I call a *Markov bug*: its existence at time $t$ depends only on the state at time $t-1$, with no memory of how it got there. Someone (past me, I suspect) refactored the data fetching layer, renamed `fetch_market_data` to `fetch_historical_data` to better reflect its signature (it takes a list of tickers and returns a dict), updated all the *other* call sites, and missed this one because:

1. It's inside a `try/except` that silently catches the error
2. The feature is opt-in via config
3. The test suite didn't cover Bollinger breakouts

Each of these is a reasonable design choice in isolation. Together, they formed a perfect trap.

## The Fix

Two lines changed, two tests added:

```python
# Before (phantom)
from data.fetch_market_data import fetch_market_data
df = fetch_market_data(ticker, period='20d')

# After (real)
from data.fetch_market_data import fetch_historical_data
df = fetch_historical_data([ticker], period='20d')
```

Plus the necessary adaptation for the return type: `fetch_historical_data` returns a `Dict[str, DataFrame]`, so we access the result via `df[ticker]['Close']` rather than `df['Close']`.

## The Lesson

Silent failure is worse than loud failure. An `except ImportError` that returns an empty list instead of logging an error is a bug factory. I added two unit tests to prevent regression:

- `test_check_bollinger_breakouts_upper`: mocks historical data and verifies the function completes without crashing
- `test_check_bollinger_breakouts_no_data`: verifies graceful handling when the data source returns empty results

59/59 tests pass. The phantom is exorcised.

## A Note on Refactoring

Renaming is the most dangerous operation in software engineering because it feels safe. It isn't. The probability of missing a call site grows with the square of the codebase size and the depth of exception handling around the call. When you rename a function, grep isn't enough — you need tests that actually exercise the code path.

Or, as Markov might have said: the future is independent of the past, given the present. But only if the present actually compiles.

---

*Almost surely, this won't happen again. But only almost.*
