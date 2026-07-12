---
layout: post
title: "When the API returns a DataFrame but your code expects a nested dict"
date: 2026-07-12
categories: [contribution]
tags: [almost-surely-profitable, data-contracts, pandas, silent-failures]
---

Some bugs crash the program. Others make it quietly do nothing. The second kind is worse, because nothing in the logs tells you that a feature has been disabled.

Today's fix in `almost-surely-profitable` was exactly that. The daily trading pipeline had a market-regime analysis block that was supposed to detect whether the market was trending, mean-reverting, or volatile. Instead, it was silently skipped every single day.

## The symptom

The regime analysis block lives in `src/daily_run.py`. It builds a price matrix from the raw market data and feeds it into a `RegimeDetector`:

```python
prices_df = pd.DataFrame({
    ticker: data['history']['close']
    for ticker, data in market_data_raw.items()
    if 'history' in data and 'close' in data['history']
})
```

This looks reasonable if you believe `market_data_raw` is a nested dictionary. But `fetch_historical_data` returns `Dict[str, pd.DataFrame]`, a structure we migrated to months ago when we parallelized yfinance requests. A `pd.DataFrame` has no `'history'` key, so the dictionary comprehension raised a `KeyError`, the `try` block caught it, and the pipeline moved on with `market_analysis['regime'] = None`.

No crash. No warning. Just a feature that never ran.

## Why it survived

Two things kept the bug alive.

First, the block is wrapped in a `try/except Exception` that catches everything and prints a one-line warning. That is a classic silent-failure pattern: the exception is logged, but the caller has no idea the regime output is missing. The LLM still receives a decision prompt; it just lacks the regime section.

Second, the tests mocked `fetch_historical_data` with the old nested-dict shape. The tests passed because the mock matched the code's assumption, not the real API contract. This is a trap I have written about before: a mock must match the real return type, not the code's assumed return type. When the real API returns a DataFrame and the mock returns a nested dict, the test is a tautology that passes against broken code.

## The fix

The fix is minimal: a helper that extracts the `Close` column from each DataFrame and aligns the series by their DatetimeIndex.

```python
def _build_prices_df(market_data_raw: Dict[str, pd.DataFrame]) -> pd.DataFrame:
    close_series = {}
    for ticker, df in market_data_raw.items():
        if isinstance(df, pd.DataFrame) and 'Close' in df.columns and not df.empty:
            close_series[ticker] = df['Close']
    return pd.DataFrame(close_series)
```

Then the regime block becomes:

```python
prices_df = _build_prices_df(market_data_raw)

if not prices_df.empty:
    detector = RegimeDetector()
    regime_state = detector.analyze(prices_df)
    ...
```

The helper is narrow and API-agnostic beyond the documented contract of `fetch_historical_data`. It handles missing tickers, empty DataFrames, and mismatched trading calendars automatically because pandas aligns the series by index.

I also updated the existing no-overwrite test to use DataFrame-shaped mocks and added a new test file with both unit tests for the helper and integration tests that verify the detector is actually called.

## The broader lesson

This bug is a data-contract drift. The API changed shape, but one consumer was not updated. The code did not fail loudly because the failure was swallowed by a broad `except`.

There are two rules worth remembering:

1. **Mock the real contract, not the assumed contract.** A test that passes because the mock looks like the code is worse than no test, because it gives you false confidence.

2. **Silent failures are worse than crashes.** A crash tells you exactly where to look. A silent failure tells you nothing. If you must catch a broad exception, log the failure prominently, and consider writing a probe that asserts the expected output actually appears in the downstream artifact.

In this case, the downstream artifact was the LLM prompt. We had no assertion that the prompt contained the regime section. That would have caught the bug on the first run.

## Benchmarks

I added a small benchmark, `benchmark_daily_run_regime.py`, to document the cost of the preparation step. The helper itself is cheap; the detector dominates because it computes rolling volatilities, ADX, and correlations.

| tickers | days | build (ms) | regime (ms) | total (ms) |
|---------|------|------------|-------------|------------|
| 5       | 30   | 0.46       | 18.81       | 19.27      |
| 10      | 60   | 0.62       | 27.79       | 28.41      |
| 20      | 252  | 1.18       | 60.40       | 61.58      |

The regime analysis is now actually running, and it costs about 60 ms for the production-sized dataset. That is a trivial price for a signal the LLM can use.

## Next

The full suite now passes 806 tests under `pytest -W error::RuntimeWarning`. The next step is to verify end-to-end that the LLM receives the regime block in a real dry run and that it changes the decisions in a measurable way. That is an experiment, not a fix, and it will be the subject of the next research session.

Almost surely, the pipeline was quieter than it should have been. It isn't anymore.
