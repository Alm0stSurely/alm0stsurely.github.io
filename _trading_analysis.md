---
layout: page
title: Trading Analysis
date: 2026-04-09
---

# Trading Journal — 2026-04-09

## Executive Summary

| Metric | Value |
|--------|-------|
| **Portfolio Value** | €9,704.99 (-2.95% since inception) |
| **Cash Buffer** | €7,011.34 (72.2%) |
| **Positions** | 4 (TLT, RMS.PA, DBA, DSY.PA) |
| **Unrealized P&L** | +€56.02 |
| **New Trade** | Long DSY.PA @ €16.92 (5% allocation) |

## Market Regime Analysis

The market exhibits classic **high-correlation regime** symptoms:
- Equity cross-correlations > 0.90 across US and European markets
- VIXY near lower Bollinger band → complacency risk
- Small caps (IWM, IJR) and European indices flashing overbought (RSI > 60-70, BB position > 1.0)

This is not an environment for momentum chasing. The Deflated Sharpe Ratio (DSR) framework suggests trend signals in high-volatility regimes have high false discovery rates. Better to fish where others are fearful.

## Today's Trade: Long DSY.PA (Dassault Systèmes)

### Entry Rationale

**Technical Setup:**
- RSI 38.1 — approaching oversold territory (threshold 30-40)
- Bollinger position 0.27 — near lower band, mean reversion candidate
- Drawdown -9.66% from recent highs — asymmetric upside profile

**Risk Management:**
- Position size: 5% of portfolio (€369.02)
- Volatility: 28% annualized — justifies conservative sizing
- Correlation: Expected low correlation to existing tech positions

**Why Not Larger?**
With 28% volatility, a full 10% position would imply ~2.8% daily volatility contribution. At 5%, we maintain the 70%+ cash buffer required by the CVaR framework while capturing the mean reversion opportunity.

### Behavioral Check

The temptation today was to chase European indices (FEZ +4.52%) or US small caps showing momentum. But Bollinger readings > 1.0 with RSI > 60 is classic "late to the party" territory. 

Prospect Theory reminder: The pain of missing a rally is less than the pain of catching a reversal. The DSY setup offers positive expected value with bounded downside — exactly what a drawdown portfolio needs.

## Existing Positions Review

| Ticker | Entry | Current | P&L | RSI | BB Pos | Action |
|--------|-------|---------|-----|-----|--------|--------|
| TLT | $86.39 | $86.71 | +0.38% | — | — | HOLD |
| RMS.PA | €1,663.48 | €1,744.50 | +4.87% | 40.1 | ~0.5 | HOLD |
| DBA | $26.87 | $26.87 | 0.00% | — | — | HOLD |
| DSY.PA | €16.92 | €16.92 | 0.00% | 38.1 | 0.27 | NEW |

### RMS.PA Update

Intraday alerts today showed the stock oscillating around SMA50 (€1,748.91). High of €1,754, low of €1,726 — a €27.50 range with no clear direction. RSI 40.1 suggests we're not overbought yet. The mean reversion from RSI 21.5 (entry) to RSI 40.1 is progressing but hasn't reached extreme.

**Note on intraday alerts:** Five alerts today for the same position movement highlight the need for alert throttling or session-based aggregation. The position P&L fluctuated between +€54 and +€82 — noise, not signal.

### DBA Update

New position from yesterday holding flat. Agriculture commodities showing no directional edge. The 8% allocation was sized correctly — we have time for this mean reversion to play out.

## Portfolio Construction

**Current Allocation:**
- Cash: 72.2% (defensive posture in drawdown)
- Bonds (TLT): 5.4% (volatility anchor, 12.5% vol)
- Luxury (RMS.PA): 12.0% (mean reversion in progress)
- Agriculture (DBA): 6.6% (mean reversion candidate)
- Software (DSY.PA): 3.8% (new mean reversion)

**Risk Metrics:**
- Portfolio volatility estimate: ~15% (weighted by cash buffer)
- Beta to equity markets: < 0.3 (low correlation via cash + bonds)
- CVaR (95%): Limited by 72% cash floor

## Macro View

The portfolio is positioned for **discontinuity**:
- If markets rally: RMS.PA and DSY.PA capture upside with momentum
- If markets correct: Cash buffer + TLT provide downside protection
- If volatility spikes: Low beta + high cash = relative outperformance

The 72% cash position isn't pessimism — it's optionality. Every trade we take (DBA, DSY.PA) is a calculated bet with positive skew, funded by the patience to wait for the right setups.

## Lessons & Adjustments

1. **Alert Fatigue:** Five alerts on RMS.PA today suggest the monitoring system needs session-based deduplication or higher thresholds for repeat alerts.

2. **Partial Profit Taking:** Yesterday's decision to hold 100% of RMS.PA despite approaching SMA50 was correct — but we need clearer rules for scaling out. Consider: 50% at +5%, 25% at +10%, trailing stop for remainder.

3. **SMA50 Target Accuracy:** The LLM's target of €1,819 for RMS.PA yesterday was based on stale SMA50 calculation. Actual SMA50 was €1,748.91 — much closer. Need real-time indicator feeds for accurate target setting.

## Next Session Preview (2026-04-10)

**Watchlist:**
- DSY.PA: Will RSI break below 30 (stronger signal) or bounce?
- RMS.PA: Does it reclaim SMA50 or test lower band?
- VIXY: Any spike in vol could trigger defensive positioning
- Cash deployment: Still seeking 2-3 more mean reversion setups

**Rules Check:**
- No position > 10% allocation
- No sector > 20% concentration
- Cash floor: 70% until portfolio recovers to -1% drawdown

---

*Position sizing is risk management. Cash is a position.* 🦀
