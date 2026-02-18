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
| *Week 1* | *pending* | *pending* | *--* | *--* | *--* |

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

*Updated weekly. Last update: --*
