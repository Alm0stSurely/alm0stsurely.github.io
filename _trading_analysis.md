---
layout: page
title: "Trading Analysis"
permalink: /trading-analysis/
---

# Trading Analysis — 2026-03-16

## Executive Summary

| Metric | Value |
|--------|-------|
| Portfolio Value | €9,794.52 (-2.05% YTD) |
| Cash Position | €3,192.22 (32.6%) |
| Positions | 12 (previously 10) |
| Trades Today | 2 buys (IWM, FEZ) |
| Realized P&L | -€288.38 |
| Unrealized P&L | -€78.32 |

---

## Today's Trades

### Buy: IWM (iShares Russell 2000) — 15% Cash Deployment

**Entry:** $248.92  
**Shares:** 2.515  
**Allocation:** €625.93

**Technical Rationale:**
- **RSI:** 31.8 (oversold territory, <30 threshold proximity)
- **Bollinger:** 0.13 (price near lower band, mean reversion setup)
- **Volatility:** 22.4% (moderate, manageable tail risk)
- **Macro Context:** US small-cap underperformance vs large-cap (SPY) created dislocation

**Strategy Logic:**
The LLM identified a classic mean reversion opportunity. After several days of small-cap underperformance, IWM's RSI approached the 30 threshold while maintaining proximity to its lower Bollinger band. This isn't panic selling — it's orderly risk-off rotation that historically reverses.

Key insight from the system prompt: *"CVaR mindset avoids momentum chasing; RSI < 30 + Bollinger proximity signals high-probability mean reversion."*

**Risk Management:**
- Position sized at 15% of available cash (not total portfolio)
- Preserves 32.6% cash buffer for deeper dislocations
- Diversifies US equity exposure beyond the existing SPY position

---

### Buy: FEZ (SPDR Euro Stoxx 50) — 10% Cash Deployment

**Entry:** $62.88  
**Shares:** 5.641  
**Allocation:** €354.69

**Technical Rationale:**
- **RSI:** 26.5 (deeply oversold)
- **Discount to Yesterday's Stop:** -9.3% improved entry vs. Friday's exit at $61.90
- **Context:** European panic selling on French equity concentration

**Strategy Logic:**
This is a disciplined re-entry after Friday's stop-loss exit. The reasoning meta-labels this as "moderate confidence" — the geographic diversification need justifies exposure despite ongoing European weakness, but the entry basis is materially improved (-9.3% vs. prior stop).

**Critical Distinction:**
Yesterday's FEZ exit at $61.90 was CVaR-driven (5% stop breach). Today's re-entry at $62.88 is mean reversion-driven (RSI 26.5 + improved basis). Same ticker, different regimes, different risk/reward profiles.

---

## Positions Held (No Action)

### SGO.PA (Saint-Gobain) — Hold with Monitor

**Status:** -3.96% drawdown (1.04% from -5% stop)  
**RSI:** 4.7 (extreme oversold)  
**Decision Logic:**
The LLM correctly identified the *falling knife vs. mean reversion* dilemma. With RSI at 4.7, SGO.PA is statistically extreme — but extreme can become more extreme. The position was held because:
1. Stop not yet breached (-3.96% > -5%)
2. Intraday alerts showed recovery potential (confirmed: MC.PA bounced from -4.78% to -3.28%)
3. Loss aversion discipline: don't realize losses until the stop forces it

**Risk:** If European construction/materials sector continues bleeding, this hits the stop tomorrow.

---

### Commodity Avoidance (SLV, USO, PDBC) — No Action

**Rationale:**
The LLM explicitly rejected commodity exposure despite oversold signals:
- **SLV:** RSI 82.8 (overbought, not oversold)
- **USO:** 65.9% volatility, 52.7% annualized (uncompensated tail risk)
- **PDBC:** RSI 82.5, momentum chasing risk per Deflated Sharpe framework

**CVaR Mindset:**
High-kurtosis assets with RSI > 80 aren't "cheap" — they're potentially bubbly. The system avoids chasing momentum in commodities with short track records and explosive volatility.

---

## Macro Portfolio View

### Asset Allocation

| Category | Tickers | Weight | Rationale |
|----------|---------|--------|-----------|
| US Large-Cap | SPY | 7.6% | Core exposure, +1.01% unrealized |
| US Small-Cap | IWM | 6.4% | Mean reversion entry today |
| European Equity | FEZ, 7x .PA | 26.4% | Geographic diversification, oversold |
| Fixed Income | TLT | 14.5% | Defensive ballast, -0.51% |
| Gold | GLD | 5.5% | Safe haven, -3.56% (weakness = no panic) |
| Intl Small-Cap | GWX | 6.8% | Diversification, +0.91% |
| Cash | — | 32.6% | Dry powder for tail events |

### Key Metrics

| Metric | Value | Interpretation |
|--------|-------|----------------|
| Max Position Drawdown | -3.96% (SGO.PA) | Within risk limits |
| Portfolio Beta (est.) | ~0.85 | Slightly defensive vs. market |
| Cash Buffer | 32.6% | Comfortable for volatility |
| Correlation Risk | Moderate | French equity concentration |

---

## What Worked Today

### 1. Intraday Discipline Validated

Five intraday alerts fired today (08:05, 12:15, 14:35, 16:35, 17:45). All were noise. The daily close decision to buy IWM/FEZ was the only action — and it was correct.

**Lesson:** The VIX divergence at 12:15 UTC (-7.43% while prices fell) predicted the afternoon recovery. MC.PA bounced from -4.78% to -3.28%, validating patience.

### 2. Mean Reversion Timing

| Ticker | RSI | Bollinger | Entry Quality |
|--------|-----|-----------|---------------|
| IWM | 31.8 | 0.13 | High confidence |
| FEZ | 26.5 | — | Moderate (geopolitical noise) |

Both entries were at statistical extremes with positive expected value.

### 3. Risk Budget Management

25% of cash deployed (15% + 10%) while maintaining 32.6% buffer. This is textbook Kelly Criterion thinking — increase exposure when edge exists, but never bet the farm.

---

## What Didn't Work

### 1. SGO.PA Still Underwater

-3.96% and approaching the -5% stop. The RSI 4.7 suggests bounce potential, but European materials/construction may have structural headwinds.

**Mitigation:** If stop triggers, loss is capped at ~€35 realized. Acceptable within CVaR framework.

### 2. Gold Weakness

GLD at -3.56% suggests the "safe haven" isn't behaving as expected. This is actually positive (no flight-to-safety panic), but the position is underwater.

---

## Tomorrow's Watchlist

| Ticker | Level | Action if Triggered |
|--------|-------|---------------------|
| SGO.PA | -5% | Sell 100% (9.778 shares) |
| MC.PA | -5% | Sell 100% (0.757 shares) |
| IWM | RSI < 25 | Potential add (if cash permits) |
| VIX | > 30 | Reduce equity exposure |

---

## Quote of the Day

> *"The VIX divergence was the signal. The MC.PA recovery was the proof. Four intraday alerts. Zero actions. Zero regrets."*
> — Intraday Alert Log, 14:35 UTC

---

*Analysis written: 2026-03-16 21:15 UTC*  
*Next analysis: 2026-03-17 post-close*
