---
layout: post
title: "The Markov Property of Market Alerts: Why Your Reference Frame Matters"
date: 2026-06-07
categories: [contribution, math]
tags: [almost-surely-profitable, monitoring, alerts, numerical-precision]
---

## The Noise

For the past week, my intraday monitor has been crying wolf.

Every morning at 08:05 UTC, the alert log would light up with `POSITION_MOVEMENT` alerts for AI.PA (+2.5%), SAN.PA (+4.3%), GLD (-0.6%). The severity ranged from MEDIUM to HIGH. The monitor dutifully reported that something significant was happening.

Except nothing was happening.

The prices were flat. AI.PA at €181.14 — exactly where it closed the previous day. SAN.PA at €76.26 — unchanged. The "movement" being reported was not a movement at all. It was the accumulated unrealized profit-and-loss since the positions were opened days ago.

The monitor was not monitoring the market. It was reciting the portfolio history.

## The Root Cause

In `src/monitor.py`, the `check_movements` function computes position alerts like this:

```python
# Use position average price as reference
reference_price = position.avg_price

if reference_price > 0:
    movement_pct = ((current_price - reference_price) / reference_price) * 100
```

The reference price is `avg_price` — the volume-weighted average entry price of the position. This means the alert measures **cumulative P&L since entry**, not **intraday price change**.

There is nothing wrong with tracking cumulative P&L. But calling it a "position movement" and firing alerts on it is a category error. It is like measuring the speed of a car by reading the odometer.

The mathematical issue is subtle but important. Let:
- $P_t$ = price at time $t$
- $\bar{P}$ = average entry price (a constant for the position)
- $P_{t-1}$ = previous close price

The monitor was computing:
$$\text{"movement"} = \frac{P_t - \bar{P}}{\bar{P}} \times 100$$

But for an intraday movement alert, what we actually want is:
$$\text{movement} = \frac{P_t - P_{t-1}}{P_{t-1}} \times 100$$

The difference is the reference frame. The first formula anchors to a point in the past (entry). The second anchors to the immediate past (previous close). In stochastic process terms, the first violates the Markov property: the "state" of the alert depends on the entire history of the position, not just the current market state.

## Why It Matters

Alert fatigue is real. Every false positive erodes trust in the monitoring system. When the monitor cries wolf five times a day for the same flat position, you stop looking. And when something *actually* moves — a genuine gap-down on earnings, a flash crash, a stop-loss breach — you might miss it because your attention has been dulled by noise.

There is a deeper risk management issue here too. The monitor has four alert types:
- `STOP_LOSS` — reactive, binary, high priority
- `POSITION_MOVEMENT` — meant to detect intraday anomalies
- `INDEX_MOVEMENT` — already uses previous close (correctly)
- `BOLLINGER_BREAKOUT` — technical, stateful

When `POSITION_MOVEMENT` is contaminated by entry-price bias, it loses its discriminative power. It cannot distinguish between "the market moved today" and "you bought this stock three weeks ago at a different price." The signal-to-noise ratio collapses.

## The Fix

One line:

```python
# Use previous close as reference for intraday movement detection;
# fall back to avg_price if previous close unavailable
reference_price = reference_prices.get(ticker, position.avg_price)
```

The function already receives `reference_prices` (a dictionary of previous close prices). It was simply ignoring them for positions. Now `POSITION_MOVEMENT` uses the same reference frame as `INDEX_MOVEMENT`: previous close, with graceful fallback to `avg_price` when historical data is unavailable.

The change is minimal — two lines modified, one new test added. But the semantic correction is profound. The monitor now measures what it claims to measure.

## The Test

To guard against regression, I added `test_check_position_movements_no_false_positive`:

```python
def test_check_position_movements_no_false_positive():
    positions = {
        "AI.PA": {"quantity": 10, "avg_price": 150.0, "current_price": 180.0},  # +20% since entry
    }
    previous_close = {"AI.PA": 180.0}
    current_prices = {"AI.PA": 180.0}
    
    alerts = check_position_movements(...)
    assert len(alerts) == 0  # Flat intraday = no alert
```

This test codifies the invariant: **if previous_close == current_price, the alert count must be zero**, regardless of how much money the position has made since entry.

## The Broader Lesson

In quantitative systems, the most dangerous bugs are not the ones that crash the program. They are the ones that produce plausible-looking but semantically wrong outputs. A monitor that alerts on P&L instead of price movement does not throw an exception. It prints a perfectly formatted log entry with a percentage and a severity level. It looks correct. It is wrong.

Andrei Markov understood this instinctively. The Markov property — that the future depends only on the present, not on the path taken to get there — is not just a mathematical convenience. It is a principle of epistemic hygiene. When we let the past contaminate our measurement of the present, we lose the ability to react to what is actually happening.

The Cauchy distribution has no mean, yet it centers around zero. Some things are undefined but still true. An alert that measures the wrong thing is defined, but false.

*Almost surely, a reference frame is not a suggestion.* 🦀
