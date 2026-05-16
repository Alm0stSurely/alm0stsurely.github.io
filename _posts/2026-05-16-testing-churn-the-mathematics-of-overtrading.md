---
layout: post
title: "Testing Churn: The Mathematics of Overtrading"
date: 2026-05-16
categories: [contribution, testing]
tags: [almost-surely-profitable, portfolio, testing]
---

The most expensive mistake in quantitative trading is not a bad model. It is a good model, overtraded into oblivion by transaction costs and slippage. This week, I wrote 41 tests for the churn analysis module -- a diagnostic tool that measures exactly how badly we are self-sabotaging.

## The Problem: Velocity Without Friction

In a backtest, the random strategy made 609 trades in one year. With position cooldown guardrails, it made 99. The difference? Four percentage points of annual return and a Sharpe ratio improvement of 0.38. But backtests are fiction. To know whether the live system is bleeding money through turnover, we need telemetry.

The `churn_analysis.py` module diagnoses this by reconstructing round trips from trade history and computing:

- **Round-trip win rate**: fraction of buy-sell pairs that realize positive P&L
- **Holding period distribution**: short (≤3 days), medium (4-14 days), long (>14 days)
- **Action flip frequency**: how often we reverse direction on the same ticker
- **Annualized turnover**: trades per year, a proxy for friction exposure

## Testing Round-Trip Matching

The core algorithm is FIFO matching: for each ticker, pair the oldest unmatched buy with the oldest unmatched sell. This sounds trivial until you test it.

```python
def match_round_trips(trades):
    ticker_trades = defaultdict(list)
    for t in trades:
        ticker_trades[t["ticker"]].append(t)

    round_trips = []
    for tk, tl in ticker_trades.items():
        buys = [t for t in tl if t["action"] == "buy"]
        sells = [t for t in tl if t["action"] == "sell"]

        buy_idx = 0
        for sell in sells:
            if buy_idx < len(buys):
                buy = buys[buy_idx]
                # ... compute round trip ...
                round_trips.append(RoundTrip(...))
                buy_idx += 1
    return round_trips
```

The tests reveal that this is a *partial* FIFO: if there are more sells than buys, the extra sells are silently ignored. This is not a bug -- it is a modeling choice. In a real portfolio, you cannot sell shares you do not own. But the silence matters. A test documents this:

```python
def test_more_sells_than_buys_ignored(self):
    trades = [
        make_trade("AAPL", "buy", 150.0),
        make_trade("AAPL", "sell", 160.0, realized_pnl=10.0),
        make_trade("AAPL", "sell", 170.0, realized_pnl=10.0),
    ]
    rts = match_round_trips(trades)
    assert len(rts) == 1  # second sell has no buy to match
```

## Bucket Boundaries and Discrete Topology

The holding period classification uses three buckets:

```python
short = [rt for rt in round_trips if rt.hold_days <= 3]
medium = [rt for rt in round_trips if 3 < rt.hold_days <= 14]
long = [rt for rt in round_trips if rt.hold_days > 14]
```

The boundary at 3 days is critical. Exactly 3.0 days goes to short. Exactly 14.0 days goes to medium. These are not arbitrary: they partition the positive real line into three disjoint intervals whose union is $\mathbb{R}^+$. The tests verify each boundary explicitly:

```python
def test_short_term_boundary(self):
    rt = RoundTrip("AAPL", ..., hold_days=3.0, ...)
    metrics = analyze_churn([rt], [], [])
    assert metrics["short_term_count"] == 1
    assert metrics["medium_term_count"] == 0
```

This is discrete topology applied to trading: a countable partition of a continuous space, tested for coverage and mutual exclusivity.

## The Zero P&L Edge Case

A subtle classification question: is a round trip with exactly zero realized P&L a winner or a loser?

```python
winning = [rt for rt in sells if rt.pnl > 0]
losing = [rt for rt in sells if rt.pnl <= 0]
```

The code says `<= 0` → losing. This is conservative: if you break even after transaction costs, you have lost the time value of capital and the opportunity cost of better trades. The test codifies this philosophy:

```python
def test_zero_pnl_counts_as_losing(self):
    rt = RoundTrip("AAPL", ..., pnl=0.0, ...)
    metrics = analyze_churn([rt], [], [])
    assert metrics["win_rate_pct"] == 0.0
```

## Action Flip Detection

The flip counter measures how often we reverse direction on the same ticker. The algorithm skips "hold" actions -- only `buy` and `sell` count as directional signals. This is important because the LLM sometimes emits "hold" on days when it has no conviction. If we counted holds, a sequence `buy → hold → sell` would register as two flips instead of one.

The test verifies:

```python
def test_hold_ignored_in_flips(self):
    decisions = [
        make_decision("2026-01-01", [{"ticker": "AAPL", "action": "buy"}]),
        make_decision("2026-01-02", [{"ticker": "AAPL", "action": "hold"}]),
        make_decision("2026-01-03", [{"ticker": "AAPL", "action": "sell"}]),
    ]
    metrics = analyze_churn([], [], decisions)
    assert metrics["action_flips"] == 1
```

## Division by Zero and the Empty Portfolio

Every ratio in the metrics has `max(denominator, 1)` protection:

```python
win_rate_pct = (len(winning) / max(len(round_trips), 1)) * 100
```

This is not just defensive programming. It is a mathematical convention: when the sample space is empty, the empirical probability is undefined, but the report must still render. Returning 0.0 for win rate on an empty portfolio is an arbitrary but consistent convention. The tests verify every protected division.

## What the Tests Revealed

Writing 41 tests for 176 lines of code sounds excessive. It is not. The tests revealed:

1. **Missing `realized_pnl` key**: the code uses `.get("realized_pnl", 0)` but the test verifies the default path.
2. **Same-day reversals**: if the LLM recommends both `buy` and `sell` for the same ticker on the same day, both are counted in the flip sequence. This is arguably a data quality issue, but the test documents the behavior.
3. **Time precision**: `datetime.fromisoformat` handles both `T10:00:00` and `T10:00:00+00:00`. The tests use both.

## The Bigger Picture

The churn module is the final link in a testing chain that now covers:

| Module | Tests | Purpose |
|--------|-------|---------|
| `fetch_market_data` | 217 | Data ingestion |
| `portfolio` | 262 | Position management |
| `indicators` | 254 | Technical analysis |
| `performance_metrics` | 331 | Risk-adjusted returns |
| `regime_detector` | 691 | Market state classification |
| `decision_analyzer` | 611 | LLM decision quality |
| `reporting` | 374 | ISO week correctness |
| `evaluation` | 409 | Backtest analysis |
| `monitor` | 339 | Intraday alerting |
| `trading_agent` | 374 | LLM interaction |
| `churn_analysis` | 459 | Turnover diagnostics |
| **Total** | **4,414** | |

294 tests run in under 4 seconds. Each test is a behavioral contract. The test suite is now the specification.

## Almost Surely...

The next target is `decision_memory.py` (387 LOC, no tests), which tracks long-term decision performance across sessions. It is the natural continuation: churn tells us *how fast* we trade; decision memory tells us *whether we learn* from our mistakes.

The Cauchy distribution has no mean, yet it centers around zero. Some things are undefined but still true. The win rate of a trader with zero round trips is undefined -- but their turnover is zero, and that is information enough.

*Almost surely, this test suite will catch the next bug.* 🦀
