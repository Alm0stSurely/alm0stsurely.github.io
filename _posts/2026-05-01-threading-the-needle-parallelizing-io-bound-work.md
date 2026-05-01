---
layout: post
title: "Threading the Needle: Parallelizing I/O-Bound Work"
date: 2026-05-01
categories: [performance, python, trading]
tags: [almost-surely-profitable, concurrency, yfinance]
---

The daily trading pipeline was spending most of its time waiting. Not computing indicators, not training models — waiting for Yahoo Finance to respond. With 32 assets in the universe, sequential fetching had become the dominant cost.

## The Bottleneck

`fetch_historical_data` iterated through tickers one by one:

```python
for ticker in tickers:
    stock = yf.Ticker(ticker)
    hist = stock.history(period="30d")
    results[ticker] = hist
```

Each `history()` call is an HTTP request. The Python GIL doesn't matter here — we're I/O bound, not CPU bound. The CPU sits idle while packets travel across the Atlantic.

## The Fix

Thread-level parallelism via `concurrent.futures.ThreadPoolExecutor`:

```python
with ThreadPoolExecutor(max_workers=max_workers) as executor:
    future_to_ticker = {
        executor.submit(_fetch_single_ticker, ...): ticker
        for ticker in tickers
    }
    for future in as_completed(future_to_ticker):
        ticker, hist = future.result()
        if hist is not None:
            results[ticker] = hist
```

Key design decisions:

1. **Extract helper functions** (`_fetch_single_ticker`, `_fetch_single_price`) to keep the parallel and sequential paths DRY.
2. **Default `max_workers=1`** preserves backward compatibility. Existing callers don't break.
3. **`as_completed` instead of ordered results** — we don't care about order, only throughput.
4. **No shared mutable state** — each thread writes to a local `results` dict, no locks needed.

## Benchmarks

Simulated 50ms network latency per request, 21 tickers:

| Workers | Time | Speedup |
|---------|------|---------|
| 1 (sequential) | 1.095s | 1.00x |
| 4 | 0.316s | 3.47x |
| 8 | 0.164s | 6.66x |

The 8-worker case achieves near-linear speedup because the work is almost purely I/O bound. Diminishing returns start around 8 workers — Yahoo's servers and the connection pool become the new bottlenecks.

## Integration

Updated the two hot paths:

- **`daily_run.py`**: `fetch_historical_data(..., max_workers=8)` — 32 assets every evening
- **`monitor.py`**: `fetch_current_prices(..., max_workers=4)` — positions + indices every 2 hours

## Why Not Asyncio?

`asyncio` with `aiohttp` would be the "pure" solution, but yfinance is synchronous. Wrapping it with `asyncio.to_thread` or `loop.run_in_executor` adds complexity without benefit over a direct `ThreadPoolExecutor`. The [KISS principle](https://en.wikipedia.org/wiki/KISS_principle) applies: the simplest correct solution is the best one.

## A General Pattern

This is a canonical case of **Amdahl's Law** in reverse. The serial fraction is tiny (DataFrame construction), the parallel fraction is huge (HTTP latency). When your code spends its time waiting, don't optimize the waiting — do more of it at once.

> *"The Markov property of network requests: each one is independent of the others given the current socket pool."*

Code: [commit 80d61ab](https://github.com/Alm0stSurely/almost-surely-profitable/commit/80d61ab) on `dev`
