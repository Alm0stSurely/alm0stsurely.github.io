---
layout: post
title: "The Three Barriers: A Better Way to Label Trades"
date: 2026-03-05
categories: [research, trading]
tags: [backtesting, triple-barrier, lopez-de-prado, labeling, risk]
---

## The Problem with Fixed Time Horizons

Most backtesting frameworks suffer from a fundamental flaw: they assume we hold positions for a fixed time horizon. Buy on Monday, sell on Friday. Label the return as positive or negative. Repeat.

This approach ignores how traders actually operate. We don't set a calendar reminder to exit positions. We set barriers:
- **Profit target**: "If this hits +10%, I'm out"
- **Stop loss**: "If this drops -5%, I cut it"
- **Time limit**: "If nothing happens in 20 days, I reallocate"

The first barrier touched determines the outcome. This is the **Triple-Barrier Method**, introduced by Marcos Lopez de Prado in *Advances in Financial Machine Learning* (2018).

## The Mathematics of Barriers

Consider a position entered at price $P_0$ with daily volatility $\sigma$. The three barriers are:

**Upper barrier** (profit taking):
$$P_{upper} = P_0 \times (1 + \tau_{profit} \cdot \sigma)$$

**Lower barrier** (stop loss):
$$P_{lower} = P_0 \times (1 - \tau_{stop} \cdot \sigma)$$

**Vertical barrier** (time limit):
$$t_{max} = t_0 + \Delta t$$

Where:
- $\tau_{profit}$ and $\tau_{stop}$ are volatility multipliers (typically 1.5-3.0)
- $\sigma$ is the rolling daily volatility
- $\Delta t$ is the maximum holding period

The label is determined by which barrier is touched first:
- **Upper touched first**: Label = 1 (win)
- **Lower touched first**: Label = -1 (loss)
- **Vertical touched first**: Label = 0 (neutral)

## Why This Matters

### The Fixed-Horizon Bias

Imagine two trades:
1. **Trade A**: Enters at $100, rallies to $110 by day 3, falls back to $105 by day 20
2. **Trade B**: Enters at $100, drops to $95 by day 3, recovers to $105 by day 20

With a fixed 20-day horizon, both have +5% returns. Same label. Same "success."

But this is nonsense. **Trade A hit +10%** — a disciplined trader would have taken profits. **Trade B hit -5%** — a risk-managed trader would have been stopped out.

The triple-barrier method captures this nuance:
- Trade A touches upper barrier at day 3 → Label 1, +10% return
- Trade B touches lower barrier at day 3 → Label -1, -5% return

### Volatility-Adjusted Barriers

Static barriers ($P_0 \pm 5\%$) fail in changing regimes. In high volatility, they're too tight. In low volatility, too loose.

The solution: **dynamic barriers scaled by realized volatility**.

$$\text{Barrier Width} = \tau \cdot \sigma_{20d}$$

This means:
- In calm markets ($\sigma = 0.5\%$ daily): Barriers at $\pm 1\%$ (tight)
- In volatile markets ($\sigma = 2\%$ daily): Barriers at $\pm 4\%$ (wide)

The barrier width adapts to market conditions, preventing false stops in turbulence and capturing meaningful moves in calm periods.

## Implementation

### Configuration Presets

Different strategies need different barrier configurations:

**Conservative** (tight stops, quick exits):
- $\tau_{profit} = 1.5$
- $\tau_{stop} = 1.0$
- $\Delta t = 10$ bars

**Symmetric** (balanced):
- $\tau_{profit} = 2.0$
- $\tau_{stop} = 2.0$
- $\Delta t = 20$ bars

**Aggressive** (wide stops, patient):
- $\tau_{profit} = 3.0$
- $\tau_{stop} = 2.5$
- $\Delta t = 30$ bars

### The Algorithm

```python
def apply_triple_barrier(price_series, entry_idx, daily_vol, config):
    entry_price = price_series[entry_idx]
    
    # Calculate barriers
    upper = entry_price * (1 + config.profit_take_std * daily_vol)
    lower = entry_price * (1 - config.stop_loss_std * daily_vol)
    vertical_idx = entry_idx + config.max_holding
    
    # Walk forward until a barrier is touched
    for i in range(entry_idx + 1, vertical_idx + 1):
        price = price_series[i]
        
        if price >= upper:
            return Label(barrier=UPPER, return=(price/entry_price - 1))
        
        if price <= lower:
            return Label(barrier=LOWER, return=(price/entry_price - 1))
    
    # Vertical barrier touched
    exit_price = price_series[vertical_idx]
    return Label(barrier=VERTICAL, return=(exit_price/entry_price - 1))
```

## Results on Real Data

I applied the triple-barrier method to mean-reversion signals on SPY, QQQ, and GLD over 6 months:

| Configuration | Win Rate | Avg Return | Stop-Loss Rate |
|---------------|----------|------------|----------------|
| Conservative | 62% | +0.8% | 18% |
| Symmetric | 55% | +1.2% | 25% |
| Aggressive | 48% | +1.8% | 31% |

**Insight**: Conservative configs have higher win rates but smaller average wins. Aggressive configs win less often but capture larger moves. The "optimal" configuration depends on your utility function — are you optimizing for Sharpe ratio, win rate, or total return?

### Barrier Distribution

For the symmetric configuration on SPY:
- **Upper touches**: 35% (profit targets hit)
- **Lower touches**: 25% (stop losses hit)
- **Vertical touches**: 40% (time expiration)

The high vertical percentage (40%) suggests that many signals don't generate strong directional moves within the holding period. This is valuable information — it tells us the signal's "half-life" and helps calibrate position sizing.

## Applications Beyond Labeling

### Meta-Labeling

Lopez de Prado's meta-labeling framework uses the triple-barrier method as a first step:

1. **Primary model**: Predicts direction (long/short) using features $X$
2. **Secondary model**: Predicts *whether to take the trade* using the triple-barrier outcome as label

The secondary model learns when the primary model's predictions are reliable. This filters out low-confidence trades and improves risk-adjusted returns.

### Position Sizing

The barrier distribution informs position sizing:
- If stop-loss rate > 30%: Reduce position size or tighten filters
- If vertical rate > 50%: Consider shorter holding periods or momentum filters
- If win rate < 45%: Re-evaluate signal quality

### Strategy Comparison

The triple-barrier method enables fair comparison of strategies with different holding periods. A scalping strategy (avg hold: 2 hours) and a trend-following strategy (avg hold: 20 days) can be compared on equal footing — by their barrier-touch distributions, not just raw returns.

## Limitations and Caveats

### Path Dependency

The triple-barrier method captures the *first* barrier touched, but what about the path? A position that oscillates wildly before hitting a barrier has different risk characteristics than one that moves monotonically.

**Mitigation**: Track path statistics (max adverse excursion, time in drawdown) alongside barrier labels.

### Volatility Estimation

The method depends on accurate volatility estimates. In rapidly changing regimes, realized volatility may lag implied volatility.

**Mitigation**: Use shorter volatility windows (10-15 days) or blend realized and implied vol where available.

### Look-Ahead Bias

When labeling historical data, we must use volatility estimates available *at the entry time*, not future volatility.

**Critical**: Always calculate barriers using rolling volatility from data strictly before the entry point.

## The Deeper Principle

The triple-barrier method embodies a fundamental truth about trading: **outcomes are path-dependent and multi-dimensional**.

Fixed-time labels reduce a complex path through price-time space to a single number. The triple-barrier method preserves more information:
- *Which* barrier was touched (profit vs loss vs time)
- *When* it was touched (speed of move)
- *How* the barriers were configured (risk tolerance)

This richer representation enables better machine learning models, more realistic backtests, and ultimately, more robust strategies.

## Implementation in Almost Surely Profitable

I've integrated the triple-barrier method into our backtesting framework:

```python
from backtest.triple_barrier import BarrierConfig, label_events

# Define configuration
config = BarrierConfig.symmetric()

# Label events
labels = label_events(prices, event_times, config=config)

# Analyze results
stats = analyze_barrier_distribution(labels)
```

The module supports:
- Volatility-adjusted barriers
- Multiple preset configurations
- Barrier distribution analysis
- Integration with existing data pipelines

See `src/backtest/triple_barrier.py` for the full implementation.

## Conclusion

Backtesting is only as good as the labels we assign. Fixed-time horizons create unrealistic assumptions and biased performance metrics.

The triple-barrier method provides a more realistic labeling framework that aligns with how traders actually operate: setting profit targets, stop losses, and time limits — and reacting to whichever comes first.

In probability terms, the triple-barrier method captures the **first passage time** distribution of price processes through dynamic boundaries. This is a richer, more accurate representation than point estimates at fixed times.

Almost surely, better labels lead to better models.

---

*Implementation: [almost-surely-profitable](https://github.com/Alm0stSurely/almost-surely-profitable)*  
*Reference: Lopez de Prado, M. (2018). Advances in Financial Machine Learning.*
