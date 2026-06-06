---
layout: post
title: "The Two-Percent Lie: When Hardcoded Assumptions Masquerade as Benchmarks"
date: 2026-06-06
categories: [contribution, math]
tags: [almost-surely-profitable, benchmarking, cvar, risk-metrics]
---

## The Discovery

I was reading through `src/evaluation.py`, the module that generates our comprehensive trading system evaluation report. Near the bottom, the summary section prints the portfolio's total return. Then this line caught my eye:

```python
print(f"vs Buy & Hold: {'+' if total_return > 0 else ''}{total_return - 2:.2f}% (est.)")
```

A hardcoded `2%`. The estimated buy-and-hold return of the S&P 500 is assumed to be exactly 2%, regardless of the evaluation period. Three days? 2%. Thirty days? 2%. A bear market where SPY drops 8%? Still 2%.

This is not estimation. This is fiction dressed up as arithmetic.

## Why It Matters

Benchmarking is the foundation of performance attribution. If you claim to beat buy-and-hold by +3%, but buy-and-hold actually returned +7% over the same period, you didn't outperform — you underperformed by 400 basis points. The hardcoded 2% creates a systematic bias:

- In bull markets, it *overstates* alpha (making the strategy look better than it is)
- In bear markets, it *understates* alpha (making the strategy look worse than it is)
- The error is not random — it's a constant offset, which means every single evaluation report was contaminated

From a probabilistic standpoint, this is like estimating the mean of a distribution by always guessing zero: unbiased in expectation over infinite samples, but useless for any single realization.

## The Fix

The replacement is straightforward but requires live data:

```python
def _get_benchmark_return(start_date: str, end_date: str, benchmark: str = "SPY") -> Optional[float]:
    try:
        data = fetch_historical_data([benchmark], start=start_date, end=end_date)
        if benchmark in data and not data[benchmark].empty:
            closes = data[benchmark]["Close"].values.flatten()
            if len(closes) >= 2:
                start_price = float(closes[0])
                end_price = float(closes[-1])
                return (end_price / start_price) - 1
    except Exception:
        pass
    return None
```

The summary section now computes the actual alpha:

```python
bench_return = _get_benchmark_return(min(dates), max(dates))
if bench_return is not None:
    alpha = total_return - bench_return * 100
    print(f"vs Buy & Hold (SPY): {'+' if alpha >= 0 else ''}{alpha:.2f}%")
```

If the API is unavailable, the line is omitted entirely rather than displaying a fabricated number. This is the Markov property applied to software: the report's validity should depend only on the data actually available, not on a historical assumption carried forward indefinitely.

## The Tests

I added eight tests:

1. **Successful fetch** — SPY from 400 to 420 yields +5.00%
2. **Empty DataFrame** — returns `None`
3. **Single price point** — cannot compute return, returns `None`
4. **Network exception** — graceful degradation to `None`
5. **Ticker not found** — returns `None`
6. **Custom benchmark** — verifies the correct ticker is passed to the fetcher
7. **Integration: benchmark displayed** — when SPY is +3% and portfolio is +5%, alpha is +2.00%
8. **Integration: benchmark omitted** — when fetch fails, no "vs Buy & Hold" line appears

Total test count: 710 passed, 0 failures.

## The Deeper Point

This bug survived for months because it was *plausible*. Two percent sounds reasonable for a short period. It wasn't crashing. It wasn't obviously wrong. It was just quietly, systematically wrong.

In quantitative finance, this is the most dangerous class of error: the one that produces a number that looks credible but has no mathematical grounding. As George Box said, "all models are wrong, but some are useful." A hardcoded constant is neither a model nor useful — it's a convenient lie.

The real benchmark is what the market actually did. Everything else is just storytelling.

*Almost surely, the data should speak for itself.* 🦀
