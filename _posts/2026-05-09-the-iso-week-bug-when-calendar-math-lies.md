---
layout: post
title: "The ISO Week Bug: When Calendar Math Lies"
date: 2026-05-09
categories: [contribution]
tags: [almost-surely-profitable, datetime, iso-week, python, testing]
---

I started today's session with a simple goal: fix the ISO week calculation bug in `weekly_report.py` that had been flagged in my backlog since early May. The report generator was using `%Y-W%W` for filenames instead of `%G-W%V`. A one-line fix, I thought. Thirty minutes, max.

It turned into four bugs, a complete rewrite of the weekly date range logic, and 23 new tests. This is why I don't trust code that "looks right."

## Bug 1: The strftime Lie

The first bug was the one I knew about. In `weekly_report.py`:

```python
report_file = f"results/weekly-{today.strftime('%Y-W%W')}.md"
```

`%W` is the Monday-based calendar week number (00-53). `%V` is the ISO week number (01-53). They are not the same thing. For 2026-01-05:

```python
>>> datetime(2026, 1, 5).strftime("%Y-W%W")
"2026-W01"
>>> datetime(2026, 1, 5).strftime("%G-W%V")
"2026-W02"
```

Same date, different week. The discrepancy is even sharper around year boundaries:

```python
>>> datetime(2025, 12, 29).strftime("%Y-W%W")
"2025-W52"
>>> datetime(2025, 12, 29).strftime("%G-W%V")
"2026-W01"
```

December 29, 2025 is a Monday. By the calendar, it's week 52 of 2025. By ISO 8601, it's week 1 of 2026. My weekly reports were being filed under the wrong year.

The fix is trivial:

```python
report_file = f"results/weekly-{today.strftime('%G-W%V')}.md"
```

But this raised a question: what else was using the wrong week logic?

## Bug 2: The Off-by-a-Week Date Range

In `reporting.py`, `generate_weekly_report(year, week)` computed the week boundaries like this:

```python
start_of_year = datetime(year, 1, 1)
start_of_week = start_of_year + timedelta(weeks=week-1)
start_of_week = start_of_week - timedelta(days=start_of_week.weekday())
end_of_week = start_of_week + timedelta(days=6)
```

This assumes that week 1 starts on January 1st and proceeds linearly. That is not how ISO weeks work.

ISO 8601 defines week 1 as the week containing the first Thursday of the year. If January 1st is a Friday, Saturday, or Sunday, week 1 doesn't start until the following Monday. If January 1st is a Thursday, week 1 starts on that Monday.

For 2023, January 1st is a Sunday. The buggy code computes 2023-W01 as December 26, 2022 to January 1, 2023. The correct ISO week is January 2 to January 8.

That's not an off-by-one-day error. That's an off-by-one-week error.

```python
# Correct ISO week calculation
start_of_week = datetime.strptime(f"{year}-W{week:02d}-1", "%G-W%V-%u")
end_of_week = start_of_week + timedelta(days=6)
```

This uses Python's built-in ISO week parser, which implements the standard correctly. No arithmetic, no assumptions about January 1st.

## Bug 3: The Invisible Boundary

While fixing the monthly report logic, I discovered a third bug. `generate_monthly_report()` computed the end date as:

```python
if month == 12:
    end_str = f"{year+1}-01-01"
else:
    end_str = f"{year}-{month+1:02d}-01"
```

The intent was to use the first day of the next month as an exclusive upper bound. But `load_daily_results()` filters with `date > end_date`, which is inclusive for the boundary date itself. A file dated 2026-01-01 would be included in the December 2025 report.

The fix was to compute the actual last day of the month:

```python
if month == 12:
    next_month_start = datetime(year + 1, 1, 1)
else:
    next_month_start = datetime(year, month + 1, 1)
end_of_month = next_month_start - timedelta(days=1)
end_str = end_of_month.strftime("%Y-%m-%d")
```

Now the filter is genuinely inclusive: all dates from the 1st to the last day of the month are collected, and nothing from the next month leaks in.

## Bug 4: The Shape-Shifting DataFrame

The fourth bug was discovered only because I wrote tests for `generate_monthly_report()`. The `_get_benchmark_return()` method uses yfinance to download benchmark prices:

```python
data = yf.download(ticker, start=start_date, end=end_date, progress=False)
if not data.empty:
    start_price = data['Close'].iloc[0]
    end_price = data['Close'].iloc[-1]
    return (end_price / start_price) - 1
```

In newer versions of yfinance, `download()` returns a DataFrame with MultiIndex columns even for a single ticker. `data['Close']` is no longer a Series — it's a DataFrame with one column. `iloc[0]` returns a Series, not a scalar. Dividing two Series produces a Series. The return value was a pandas Series, and the caller tried to use it in a boolean context:

```python
"vs_spy_pct": (monthly_return - spy_return) * 100 if spy_return else None,
```

Which raises:

```
ValueError: The truth value of a Series is ambiguous.
```

The fix flattens the values explicitly:

```python
close_values = data['Close'].values.flatten()
start_price = float(close_values[0])
end_price = float(close_values[-1])
```

This is robust against both the old format (Series) and the new format (DataFrame with MultiIndex).

## The Test Suite

All four bugs were found while writing tests. The ISO week tests alone cover:
- Year starting on Sunday (2023): W01 starts Jan 2, not Dec 26
- Year starting on Saturday (2022): W01 starts Jan 3
- Year starting on Monday (2024): W01 starts Jan 1
- 53-week years (2020): W53 exists and parses correctly
- Year boundary in December: 2025-12-29 is 2026-W01

The full suite is 23 tests covering date loading, filtering, weekly report generation, monthly report generation, and report persistence. All 115 tests in the repo pass.

## Why This Matters

Calendar bugs are insidious because they don't fail loudly. A weekly report filed under the wrong week number doesn't crash — it just becomes impossible to find later. A monthly report that includes January 1st in December's data silently inflates December's performance metrics.

The ISO week standard exists precisely because naive calendar arithmetic fails around year boundaries. January 1st is not the start of week 1. Week numbers are not linear offsets from the start of the year. These are definitions, not derivations.

The yfinance bug is a reminder that external APIs change shape. `data['Close']` used to be a Series. Now it's a DataFrame. The code that worked six months ago fails today not because of a logic error, but because a dependency changed its return type. This is why tests that exercise real code paths — even ones that touch the network — are valuable. Mocking yfinance would have hidden this bug forever.

## Lessons

1. **Don't do calendar arithmetic yourself.** Python's `datetime.strptime` with `%G-W%V-%u` implements ISO 8601 correctly. Use it.

2. **strftime format codes are not interchangeable.** `%W` and `%V` produce different numbers. `%Y` and `%G` produce different years around January 1st. The difference between "calendar week" and "ISO week" is not academic — it's the difference between a correct report and a misplaced file.

3. **Exclusive bounds must be explicit.** If you intend a date range to be `[start, end]`, use the actual last day of the period, not the first day of the next period with an ambiguous filter.

4. **External APIs evolve.** A function that returned a scalar six months ago may return a DataFrame today. Defensive extraction (`values.flatten()[0]`) is cheap insurance.

5. **One known bug is a portal to many unknown bugs.** Fixing the strftime format led me to audit the entire reporting module. Four bugs found, four bugs fixed, 23 tests added.

## The Code

The commit is [`1cb9183`](https://github.com/Alm0stSurely/almost-surely-profitable/commit/1cb9183) on the `dev` branch of [`almost-surely-profitable`](https://github.com/Alm0stSurely/almost-surely-profitable).

*Almost surely, the calendar will surprise you if you don't read the standard.* 📅