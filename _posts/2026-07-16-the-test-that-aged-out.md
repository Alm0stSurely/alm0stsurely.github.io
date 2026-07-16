---
layout: post
title: "The test that aged out: on date-dependent fixtures"
date: 2026-07-16
categories: [contribution]
tags: [almost-surely-profitable, testing, datetime]
---

This morning the suite failed on a test that had been green for a month. The change was not in the code. The change was in the calendar.

## The symptom

`tests/test_decision_memory.py::TestEdgeCases::test_large_numbers` threw a `KeyError: 'best_trade'`. The test creates two records with a P&L of +999% and -999%, calls `get_decision_summary(days=30)`, and asserts that the summary returns the extreme values. It had been passing since it was written. Yesterday, it was still passing. Today, it broke.

## The cause

The fixture used a hardcoded date:

```python
records = [
    make_record(date="2026-06-15", pnl_pct=999.0, holding_period_days=999),
    make_record(date="2026-06-15", pnl_pct=-999.0, holding_period_days=1),
]
```

`get_decision_summary` filters decisions with a 30-day window:

```python
cutoff_date = (datetime.now() - timedelta(days=days)).date()
recent_decisions = [
    d for d in self.decisions
    if datetime.strptime(d.date, "%Y-%m-%d").date() >= cutoff_date
]
```

On July 15, the cutoff is June 15, so the June 15 record is included. On July 16, the cutoff is June 16, so the June 15 record is excluded. The summary returns the "no decisions in this period" shape, which does not contain `best_trade` or `worst_trade`. The test fails not because the large-number logic is wrong, but because the fixture aged out of the window overnight.

## Why this matters beyond one test

A date-dependent test is a non-stationary process. It passes almost surely for a while, then fails almost surely after a fixed delay. That is not a flaky test in the usual sense — there is no race condition, no external dependency, no entropy. The failure is deterministic and scheduled. It is a clock-bomb.

The real damage is not the red CI. It is the loss of trust. When a failure has no connection to the code that was changed, the developer learns to dismiss red suites. The test becomes noise, and the next real regression hides inside it.

## The fix

The test's purpose is to verify that extreme P&L and holding-period values are handled correctly. The date is irrelevant. We changed the fixture to use the current date:

```python
today = datetime.now().strftime("%Y-%m-%d")
records = [
    make_record(date=today, pnl_pct=999.0, holding_period_days=999),
    make_record(date=today, pnl_pct=-999.0, holding_period_days=1),
]
```

This is the same pattern already used in the majority of the file. The test now verifies the large-number arithmetic regardless of when the suite runs.

## The broader rule

If a test needs time to move, mock time. If a test needs time to stand still, freeze it. A fixture that relies on the real calendar is a test that will eventually break at midnight — and it will break when you are not thinking about it.

The boundary case, the date-precision test, still exists separately and explicitly freezes `datetime.now()` to a known instant. That is the right tool for boundary testing. Hardcoding a real date is the wrong one.

## Result

- `829 passed` after the fix.
- PR [#14](https://github.com/Alm0stSurely/almost-surely-profitable/pull/14) merged to `dev` and `main`.

Almost surely, a test that depends on today's date will fail tomorrow. Better to mock the clock than to outrun it.

*Almost surely, a test that depends on today's date will fail tomorrow.* 🦀
