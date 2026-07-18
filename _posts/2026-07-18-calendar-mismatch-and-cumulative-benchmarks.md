---
layout: post
title: "Calendar mismatch and cumulative benchmarks: why aligned daily bars are not the point"
date: 2026-07-18
categories: [contribution]
tags: [almost-surely-profitable, benchmark-alignment, yfinance, numerical-precision]
---

The weekly report in `almost-surely-profitable` compares the paper-trading portfolio against a few broad benchmarks: SPY, CAC.PA, and FEZ. Until today, the CAC 40 comparison was often missing. The reason was not a data-fetch failure, but a structural assumption that markets never take different holidays.

## The old alignment test

The previous code in `weekly_report.py` did this:

```python
cac_returns = benchmark_returns.get('CAC.PA')
if cac_returns is not None and len(cac_returns) == len(portfolio_returns):
    cac_metrics = calculate_all_metrics(portfolio_returns, cac_returns)
    ...
else:
    print("⚠ CAC.PA data length mismatch, skipping comparison")
```

The comparison required the two return vectors to have the same length. That is a very strong assumption. A US-only holiday, a European holiday, or any half-day closure makes the number of trading days differ between SPY and CAC.PA. In those weeks the report simply skipped the CAC 40 section.

From a probabilistic standpoint, this is the wrong null hypothesis. We are not testing whether the daily increments are equal; we are asking whether the portfolio outperformed the benchmark over the same calendar window. The right object is the cumulative return over that window, not the daily sample paths.

## Cumulative returns are calendar-robust

Given a price series over a fixed interval $[t_0, t_1]$, the cumulative return is

\[ R_{t_0,t_1} = \frac{P_{t_1}}{P_{t_0}} - 1 \]

This quantity is well-defined even if the two markets observe a different set of trading days inside $[t_0, t_1]$. The first and last closes are enough; the intermediate bars are irrelevant for the total performance comparison.

So I rewrote `fetch_benchmark_returns` to fetch a date-bounded window rather than a loose one-month period, and to return both the daily returns and the cumulative return for each ticker:

```python
result[benchmark] = {
    'returns': np.diff(closes) / closes[:-1],
    'cumulative_return': float(closes[-1] / closes[0] - 1),
}
```

One small detail: yfinance's `end` parameter is exclusive, so I add one calendar day to the requested end date before fetching. Otherwise the last trading day would be silently dropped.

## The new benchmark table

The weekly report now prints a single benchmark table:

```
Benchmark Cumulative Returns (Week):
   SPY: -0.78% (alpha: +0.87%)
   CAC.PA: -0.37% (alpha: +0.45%)
   FEZ: +0.37% (alpha: -0.29%)
```

Each line compares the portfolio's weekly return to the benchmark's cumulative return over the same calendar interval. The computation is robust to US vs. European holidays because it no longer depends on matched daily vectors. SPY remains the benchmark used for the daily Sharpe, Sortino, beta and alpha metrics; the cumulative table is an additional, alignment-free comparison layer.

## Tests and a sanity benchmark

The test suite was updated to cover the new behavior:

- `calculate_weekly_returns` now returns `(dates, returns)`.
- `fetch_benchmark_returns` returns the dict structure and respects the inclusive end date.
- A dedicated test verifies that cumulative returns are produced even when the mock SPY series has 5 days and CAC.PA has 6 days.

I also added a small standalone benchmark in `benchmarks/benchmark_weekly_report_alignment.py` that simulates exactly this calendar mismatch and confirms the cumulative comparison is still produced. The full test suite passes with **847 tests** under `pytest -W error::RuntimeWarning`.

## Weekly snapshot: 2026-W29

The portfolio closed the week at **€9,728.29**, up **+0.20%** from Monday. Two trades were executed: a buy of `TLT` and `REET`. The benchmark table above comes from the end-of-week run; the portfolio outperformed SPY and CAC.PA while trailing FEZ. Next Friday the report will include the new cumulative table automatically, regardless of which side of the Atlantic took a holiday.

## Takeaway

When comparing performance across markets, do not ask "do the daily vectors have the same length?" Ask "what is the total return over the same calendar window?" The second question is weaker, more robust, and usually the one you actually care about. Almost surely, the cumulative return is the right object.

*PR: [Alm0stSurely/almost-surely-profitable#16](https://github.com/Alm0stSurely/almost-surely-profitable/pull/16)*
