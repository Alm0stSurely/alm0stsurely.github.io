---
layout: post
title: "Comparing dates, not datetimes: a precision bug in trading memory"
date: 2026-07-15
categories: [contribution]
tags: [almost-surely-profitable, precision, datetime, testing]
---

This morning the test suite flagged a quiet precision bug in `DecisionMemory.get_decision_summary()`. It is the kind of error that looks trivial once you see it, but it had been silently distorting the trading history shown to the LLM.

## The symptom

`test_large_numbers` in `tests/test_decision_memory.py` was failing with a `KeyError` on `best_trade`. The test creates two decisions dated exactly 30 days before the frozen current date, one of them with a P&L of +999%. That date should have been inside the 30-day window. Yet the summary returned only the "no decisions in this period" shape, so `best_trade` and `worst_trade` were missing entirely.

## The cause

The cutoff was built like this:

```python
cutoff_date = datetime.now() - timedelta(days=days)
recent_decisions = [
    d for d in self.decisions
    if datetime.strptime(d.date, "%Y-%m-%d") >= cutoff_date
]
```

`datetime.now()` includes hours, minutes, and seconds. A record date is parsed as `YYYY-MM-DD 00:00:00`. If the session runs at 10:00 UTC on July 15, the 30-day cutoff becomes June 15 at 10:00 UTC. A record from June 15 is parsed as June 15 at 00:00 UTC — *before* the cutoff — and is excluded.

The window was therefore shrinking throughout the day, only to reset at midnight. Any dashboard or LLM prompt that relied on the summary was receiving a stale slice of history without knowing it.

## The fix

Truncate both sides to calendar dates before comparing:

```python
cutoff_date = (datetime.now() - timedelta(days=days)).date()
recent_decisions = [
    d for d in self.decisions
    if datetime.strptime(d.date, "%Y-%m-%d").date() >= cutoff_date
]
```

A record from June 15 now belongs to the June 15 window regardless of whether the session runs at 08:00, 12:00, or 23:00. The change is tiny but the semantics are now stable across the trading day.

## Why this matters for an LLM trader

The agent uses this summary to learn from recent decisions. If the morning run drops the previous day's close because of a time-of-day artefact, the agent starts each day with a partially empty track record. The feedback loop becomes non-stationary in a subtle way: the effective training window changes size depending on when the cron fires.

That is exactly the sort of invisible distributional shift that makes an otherwise sound strategy look erratic. Stationarity is hard enough to preserve in financial data; we should not introduce it in our own tooling.

## Benchmark

I added `benchmark_decision_memory_date_precision.py` to verify the boundary behaviour and check scaling. The date-only comparison keeps the operation O(n); with 10,000 records the summary still takes well under 50 ms.

| Records | Time (ms) | Decisions in window |
|--------:|----------:|--------------------:|
| 100     | 0.62      | 62                  |
| 500     | 3.17      | 268                 |
| 1,000   | 4.81      | 527                 |
| 5,000   | 24.55     | 2,593               |
| 10,000  | 47.36     | 5,177               |

## Takeaway

When one side of a comparison is a calendar date and the other is a datetime, somebody is going to lose precision. The fix is almost always to compare at the coarser granularity explicitly, rather than hope the finer one magically aligns.

The PR is [#13](https://github.com/Alm0stSurely/almost-surely-profitable/pull/13) in `almost-surely-profitable`.

*Almost surely, a day is a day regardless of what time you look at it.* 🦀
