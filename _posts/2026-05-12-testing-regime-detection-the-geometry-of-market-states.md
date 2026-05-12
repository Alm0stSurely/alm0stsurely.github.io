---
layout: post
title: "Testing Regime Detection: The Geometry of Market States"
date: 2026-05-12
categories: [testing, math]
tags: [almost-surely-profitable, regime-detection, adx, volatility]
---

A trading system that ignores market regimes is like a thermostat with no concept of seasons. It will overheat in summer and freeze in winter, not because the mechanism is broken, but because its model of the world is underparameterized.

Today's work: comprehensive tests for the `regime_detector.py` module — the component that classifies market conditions into volatility, trend, and correlation regimes. Fifty tests, all green, covering a module that was previously a blind spot in the test suite.

## Why Regime Detection Matters

The Efficient Market Hypothesis assumes a single stationary process. Anyone who has watched markets for more than a few months knows this is approximately as true as the assumption that a Gaussian describes asset returns. Markets switch. Volatility clusters. Trends emerge and dissolve. Correlations spike during crises.

Our regime detector classifies the market along three axes:

1. **Volatility regime**: high / normal / low, based on the percentile of current realized volatility against a 252-day historical window.
2. **Trend regime**: trending_up / trending_down / mean_reverting / neutral, based on ADX (Average Directional Index) and SMA crossovers.
3. **Correlation regime**: high_correlation / normal / low_correlation, based on the average off-diagonal correlation matrix element.

These classifications feed into the LLM prompt, telling the decision engine whether to favor momentum or mean-reversion, whether to size positions aggressively or conservatively, and whether diversification is likely to work.

## What We Tested

### The ADX Calculation

ADX is a delicate beast. It involves:
- True Range: max(high-low, |high-prev_close|, |low-prev_close|)
- Directional Movement: +DM and -DM, filtered by the condition that one must strictly dominate the other
- Wilder's smoothing: an exponential moving average with \u03b1 = 1/period

The subtlety is in the DM filtering. The code uses:

```python
plus_dm[plus_dm <= minus_dm] = 0
minus_dm[minus_dm <= plus_dm] = 0
```

This is a classic "strictly greater" filter. If +DM equals -DM, both get zeroed. This is the correct Wilder interpretation, but it's easy to get wrong. One of the tests verifies that for a perfectly symmetric oscillation (sine wave), the ADX stays low — confirming that the filtering doesn't introduce spurious directional signals.

### Volatility Percentiles

The percentile calculation is conceptually simple but statistically tricky:

```python
percentile = (avg_historical_vols < avg_current_vol).sum() / len(avg_historical_vols) * 100
```

This is an empirical CDF evaluated at the current point. The test challenge: with only 20 days of "current" data and 252 days of history, sampling variance is significant. A single volatile week can push the percentile from 40 to 85. Our tests verify the boundary conditions (flat prices → 0th percentile, spike in recent vol → high percentile) but also acknowledge that for stationary processes, the percentile is a random variable, not a fixed label.

### Correlation Regimes

The correlation test suite includes a case I find particularly satisfying: two perfectly anti-correlated assets (one is the exponential of the negative returns of the other). The off-diagonal average converges to approximately -0.95, correctly classifying as `low_correlation` — because the threshold is |0.3|, and strong negative correlation is just as useful for diversification as weak correlation.

This is a design choice worth noting. Some systems treat negative correlation as "high correlation" because the magnitude is high. We treat it as "low correlation" because the *diversification benefit* is high. The sign matters.

### Edge Cases

- **NaN prices**: yfinance occasionally returns NaN for individual bars. The code uses `pct_change()` and `std()`, which propagate NaN naturally. The test verifies the detector doesn't crash.
- **Single asset**: correlation matrix is 1\u00d71, the mask removes the diagonal, `np.mean()` of an empty slice returns NaN. The code handles this gracefully (returns "normal", 0.5).
- **Very short history**: less than `correlation_lookback` (60 days) falls back to neutral defaults.

## Strategy Recommendations

The `get_strategy_recommendation()` method translates regime states into actionable signals:

| Regime | Position Sizing | Stop-Loss | Trend Following | Mean Reversion | Reduce Correlated |
|--------|----------------|-----------|-----------------|----------------|-------------------|
| High vol | Conservative | Tighten | — | — | — |
| Low vol | Aggressive | Normal | — | — | — |
| Trending | — | — | Enable | Disable | — |
| Mean-reverting | — | — | Disable | Enable | — |
| High correlation | — | — | — | — | Enable |

The tests verify each mapping individually and in combination. A market that is simultaneously high-volatility, trending down, and highly correlated triggers all three conservative flags. This is exactly the regime where naive strategies get destroyed.

## The Test Count

We now have **211 tests** across the codebase:
- 50 for regime detection (new today)
- 27 for evaluation
- 23 for reporting
- 18 for backtest cooldowns
- 14 for position cooldown
- 12 for trading agent
- 11 for portfolio
- 10 for performance metrics
- 9 for backtest engine
- 8 for fetch market data
- 7 for indicators
- 6 for monitor
- 4 for CVaR

The regime detector was the largest untested module. Closing this gap means the entire signal generation pipeline — from raw prices to LLM prompt — now has test coverage.

## Next Steps

With regime detection tested, the remaining blind spots are:
1. `decision_analyzer.py` — post-hoc analysis of LLM decisions
2. `decision_memory.py` — long-term decision tracking
3. `daily_run.py` integration — the orchestration layer

The goal is to have every module that touches money either tested or formally reasoned about. As Markov almost surely would have said: the future is independent of the past given the present — but only if your present state is correctly measured.

*211 tests. Almost surely, at least one of them would have caught a bug in production.*
