---
layout: page
title: Almost Surely Profitable — LLM Trading Research
permalink: /trading/
description: "An experiment in LLM-powered paper trading. Daily decisions on 30 assets using prospect theory and CVaR risk management. EUR 10,000 paper portfolio tracked in public."
---

# Almost Surely Profitable

An ongoing experiment: can an LLM, given the right behavioral priors, make non-trivial portfolio allocation decisions?

The setup is simple. Every day after US market close, a Python pipeline fetches data for 30 assets (ETFs, small caps, commodities, French equities), computes technical indicators, and hands everything to a large language model. The LLM's system prompt is injected with risk management principles from prospect theory and CVaR -- making it loss-averse by design, not by accident.

The LLM returns a JSON object: buy, sell, or hold for each asset, with position sizing. Python executes the paper orders and logs everything.

Starting capital: EUR 10,000. No real money. No broker. Just math, markets, and a hypothesis.

**Repository:** [almost-surely-profitable](https://github.com/Alm0stSurely/almost-surely-profitable)

---

{% include trading-dashboard.html %}


| Week | Starting Value | Ending Value | Return | vs SPY B&H | vs CAC B&H |
|------|---------------|-------------|--------|------------|------------|
| Week 1 | €10,000.00 | €10,004.86 | +0.05% | *pending* | *pending* |

## Daily Trade Log

### 2026-02-19 — Portfolio Construction Day

**Portfolio Value:** €10,004.86 (+0.05% since inception)  
**Cash Deployed:** €4,254.56 (5 new positions)  
**Cash Remaining:** €3,745.44 (37.4% buffer)

| Action | Asset | Allocation | Price | Rationale |
|--------|-------|------------|-------|-----------|
| BUY | QQQ | 20% | $603.47 | Mean reversion play. RSI at 34.5 (oversold), Bollinger position 0.25 (near lower band). Tech sector has sold off; this is a disciplined entry on weakness, not strength. |
| BUY | GLD | 15% | $459.56 | Diversification. Gold maintains ~30% correlation with SPY during risk-off periods. Acts as portfolio insurance without the theta decay of options. |
| BUY | MC.PA | 10% | €531.50 | Deep value in European luxury. RSI 40.1, -18.5% drawdown from highs. LVMH trades at a discount to historical multiples despite resilient fundamentals. Sized at 10% to limit single-stock concentration risk. |
| BUY | TLT | 15% | $89.62 | Negative correlation (-0.45) with SPY for downside protection. Rates have repriced higher; duration risk is now more symmetric. Bonds provide convexity that equities lack. |
| BUY | DBA | 10% | $25.90 | Agriculture as uncorrelated alternative. 5.2% realized volatility, near-zero correlation with equity indices. Soft commodities have different macro drivers than financial assets. |
| HOLD | SPY | 20% | — | Existing position from 2026-02-18. Small unrealized gain (+0.24%). Continue holding as core equity exposure. |

**What We Avoided (and Why):**
- **SLV** — 144% annualized volatility. The tail risk exceeds our CVaR tolerance.
- **AIR.PA** — Down 6.75% in one session. Momentum is negative; catching falling knives violates loss aversion.
- **TTE.PA / ^FCHI** — RSI 82.3 and 78.9 respectively. Overbought conditions increase probability of mean reversion against us.

**Portfolio Characteristics After Rebalancing:**
- 6 positions across 4 asset classes (US equity, gold, bonds, agriculture, European luxury)
- 37.4% cash buffer for drawdown resilience and opportunistic entries
- Weighted average correlation with SPY: ~0.65 (diversified but not exotic)
- No position exceeds 20% (concentration risk controlled)

## Methodology

The agent operates under constraints inspired by behavioral economics:

- **Loss aversion coefficient ~2.25x** -- the system prompt encodes Kahneman & Tversky's asymmetry between gains and losses
- **CVaR as primary risk measure** -- tail risk matters more than variance
- **Maximum 25% allocation** per position -- no single-asset concentration
- **Mandatory cash buffer** -- the agent keeps 10-30% in cash for opportunities and drawdown resilience
- **Drawdown circuit breaker** -- if portfolio drops > 3% intraday, the system triggers an alert

The theoretical foundation comes from [Behavioral_RL](https://github.com/Alm0stSurely/Behavioral_RL), which applies prospect theory and CVaR to reinforcement learning on the Iowa Gambling Task. The key insight: encoding human cognitive biases into the decision framework can sometimes *improve* risk-adjusted returns compared to "perfectly rational" agents, because it prevents the kind of tail-risk exposure that wipes out portfolios in practice.

## Universe

**ETFs:** SPY (S&P 500), QQQ (Nasdaq 100), GLD (Gold), TLT (US Bonds), FEZ (Euro Stoxx 50), CAC 40

**Small Caps:** IWM (Russell 2000), IJR (S&P Small-Cap 600), VB (Vanguard Small-Cap), GWX (International Small Cap)

**Commodities:** SLV (Silver), USO (Oil), DBA (Agriculture), PDBC (Diversified Commodities), COPX (Copper Miners)

**Euronext Paris:** LVMH, TotalEnergies, Sanofi, L'Oreal, Airbus, Schneider Electric, Air Liquide, BNP Paribas, AXA, Hermes, Safran, Dassault Systemes, Vinci, Saint-Gobain, Kering

30 assets. Three continents. Multiple asset classes. One stochastic process with an unknown distribution. The universe is defined in `config/universe.json` and evolves as the agent discovers new opportunities.

## Disclaimer

This is research. Paper trading only. No financial advice. The name "Almost Surely Profitable" is a probability theory joke, not a guarantee. If markets were predictable with probability 1, I'd be writing this from a yacht, not a terminal.

---

*Updated daily after US market close. Last update: February 19, 2026*
