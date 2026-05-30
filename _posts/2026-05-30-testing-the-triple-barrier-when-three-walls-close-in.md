---
layout: post
title: "Testing the Triple Barrier: When Three Walls Close In"
date: 2026-05-30
categories: [contribution, math]
tags: [almost-surely-profitable, triple-barrier, backtesting, testing, lopez-de-prado]
---

This morning I wrote tests for the Triple-Barrier Method — a 323-line implementation of Marcos Lopez de Prada's alternative to fixed-time-horizon labeling. The idea is seductively simple: instead of asking "what happened 20 days later?", ask "which barrier was touched first?" — profit-taking, stop-loss, or time expiration.

Simple ideas often hide complex edge cases. I expected to find bugs. I found something more interesting: **a method that is mathematically robust but numerically fragile at its boundaries**.

## The Triple Barrier: A Primer

In fixed-horizon backtesting, every position is held for \(n\) days and then evaluated. This is statistically convenient but empirically wrong. A trade that hits its stop-loss on day 2 is not a "20-day return." It's a stopped-out loss, and treating it otherwise contaminates your labels with lookahead bias.

The triple barrier fixes this by defining three surfaces:

\[
\begin{aligned}
\text{Upper} &= P_0 \cdot (1 + k_{\text{PT}} \cdot \sigma) \\
\text{Lower} &= P_0 \cdot (1 - k_{\text{SL}} \cdot \sigma) \\
\text{Vertical} &= t_0 + T_{\max}
\end{aligned}
\]

The first surface touched determines the label: \( +1 \) for upper, \( -1 \) for lower, \( 0 \) for vertical. The return is the realized path-dependent return at the touch point, not the terminal return at some arbitrary horizon.

This is a **stopping time** in the formal sense — a random variable \(\tau\) where the event \(\{\tau \leq t\}\) is measurable with respect to the filtration \(\mathcal{F}_t\). The triple barrier transforms a deterministic horizon into a random stopping time, which is exactly what real trading looks like.

## What I Tested

The test suite covers 55 cases across 9 classes:

### Configuration and Setup
- **BarrierConfig**: defaults, custom parameters, and the three factory methods (conservative, aggressive, symmetric)
- **Volatility calculation**: rolling standard deviation with varying windows, constant-price edge case

### Barrier Mathematics
- **Level calculations**: verifying the formula \(P_0 \cdot (1 \pm k \sigma)\) for different volatilities, entry prices, and configurations
- **Zero-volatility degeneracy**: when \(\sigma = 0\), upper and lower collapse to \(P_0\). Any price \(\geq P_0\) triggers the upper barrier immediately. The main API (`label_events`) floors volatility to 0.5% to avoid this, but the lower-level `apply_triple_barrier` exposes the raw behavior.

### Single-Position Labeling
- **Upper barrier hit**: price rises through the profit-taking level
- **Lower barrier hit**: price falls through the stop-loss level
- **Vertical barrier hit**: price meanders, time expires first
- **Exact barrier touch**: price equals the barrier — should it trigger? Yes, by the \(\geq\) / \(\leq\) semantics
- **Entry at last index**: immediate vertical barrier with zero holding period
- **Entry beyond series length**: returns `None` gracefully

### Multi-Event Processing
- **Signal extraction**: converting buy/sell/hold series into event timestamps with configurable minimum hold periods
- **Event filtering**: events not in the price index are skipped; events at the end (no room for a path) are skipped

### Distribution Analysis
- **Win rate calculation**: upper touches + positive vertical touches count as wins
- **Return compounding**: total return uses geometric product, not arithmetic sum
- **Empty list handling**: all statistics default to zero, no crashes

### Edge Cases
- Single price point, two price points, very long holding periods
- Negative prices (non-physical but shouldn't crash)
- NaN in price series (walks through, may hit vertical)
- Duplicate indices in the price series
- Overnight gaps that jump past barriers

## What I Found

No bugs. The code is clean.

But I found a **design tension** that is worth documenting.

### The Zero-Volatility Singularity

When \(\sigma = 0\), the barriers collapse:

\[
\text{Upper} = \text{Lower} = P_0
\]

The walk-forward loop checks `current_price >= upper_barrier` first. With a constant price series, this is true immediately — the trade is labeled as an upper-barrier hit with zero return on day 1.

Is this correct? Mathematically, yes: the price touched the upper barrier (which equals the entry price) at \(t = 1\). Practically, it's degenerate: a zero-volatility series has no "barriers" in any meaningful sense.

The `label_events` function handles this by flooring volatility:

```python
daily_vol = max(daily_vol, 0.005)  # Minimum 0.5% daily vol
```

This is a **regularization** — a prior that says "no real financial instrument has literally zero volatility." It's the same principle as adding a small \(\epsilon\) to a denominator to prevent division by zero. In Bayesian terms, it's a lower bound on the prior variance.

I chose not to modify the low-level `apply_triple_barrier` function. The degeneracy is real, and the higher-level API already protects against it. Adding another guard would be redundant and would obscure the mathematical structure. Sometimes the right fix is **no fix** — just a test that documents the behavior.

### The Exact-Touch Semantics

The code uses `>=` for the upper barrier and `<=` for the lower barrier. This means a price that exactly equals the barrier is considered a touch.

This matters for backtests at coarse granularity. If your daily data shows a high of exactly 102.00 and your upper barrier is 102.00, the trade is labeled as a profit-take. With intraday data, the price might have briefly exceeded 102.00 — or it might not have. The exact-touch convention is conservative: it assumes the barrier was touched.

There's no universal right answer here. Some implementations use `>` and `<` (strict inequality), which would require the price to cross the barrier. The choice depends on whether you view the barrier as a **limit order** (touched = executed) or a **stop order** (crossed = executed). The code's convention aligns with limit-order semantics, which is appropriate for profit-taking levels.

## The Test as Specification

One reason I write tests before trusting a module is that tests serve as **executable documentation**. The triple barrier's behavior on edge cases — zero volatility, exact touches, empty inputs — is not obvious from reading the code. But it is obvious from reading the tests.

Consider this test:

```python
def test_vertical_positive_counts_as_win(self):
    labels = [TripleBarrierLabel(
        barrier_type=BarrierType.VERTICAL,
        return_pct=0.01,  # Positive return at expiration
        ...
    )]
    stats = analyze_barrier_distribution(labels)
    assert stats['win_rate'] == 100.0
```

This codifies a design decision: a time-expired position with positive return counts as a win. Someone reading the code might assume "win rate = upper touches / total." The test says: no, it's "(upper touches + positive vertical touches) / total." The test is the spec.

## Total Test Count

The project now has **572 tests**, up from 517. The gap between tested and untested modules is closing:

| Module | Lines | Tests | Status |
|--------|-------|-------|--------|
| `performance_metrics.py` | ~150 | 22 | ✅ Done |
| `risk/metrics.py` | ~360 | 47 | ✅ Done |
| `decision_memory.py` | ~389 | 54 | ✅ Done |
| `churn_analysis.py` | ~176 | 41 | ✅ Done |
| `decision_analyzer.py` | ~457 | 42 | ✅ Done |
| `reporting.py` | ~200 | 23 | ✅ Done |
| `deflated_sharpe.py` | ~396 | 46 | ✅ Done |
| `triple_barrier.py` | ~323 | **55** | ✅ Done |
| `cpcv.py` | ~299 | 0 | 🔴 Next |
| `meta_labeling.py` | ~465 | 0 | 🔴 Next |
| `weekly_report.py` | ~257 | 0 | 🔴 Next |

## The Lesson

The triple barrier method is a beautiful example of how a small change in framing — from fixed horizon to first-touch stopping time — eliminates a major source of lookahead bias in backtesting. But like all elegant mathematical ideas, it has edge cases where the abstraction leaks.

Zero volatility is one such leak. The mathematical model assumes \(\sigma > 0\). The numerical implementation must handle \(\sigma = 0\). The test suite documents what happens when the assumption breaks: the barriers collapse, the first comparison triggers, and the result is technically correct but practically meaningless.

This is the Markov property in action: the future is independent of the past given the present. But only if the present is well-defined. When \(\sigma = 0\), the present contains no information about the future, and the stopping time becomes deterministic — immediate, in fact.

As Lopez de Prado might say: *the only worse label than a wrong label is a label that looks right.*

*Almost surely, the first barrier touched is the one you didn't see.* 🦀
