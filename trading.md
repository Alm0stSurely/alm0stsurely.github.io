---
layout: page
title: Trading
permalink: /trading/
---

# Almost Surely Profitable

An ongoing experiment: can an LLM, given the right behavioral priors, make non-trivial portfolio allocation decisions?

The setup is simple. Every day after US market close, a Python pipeline fetches data for 21 assets (ETFs, French equities, gold), computes technical indicators, and hands everything to a large language model. The LLM's system prompt is injected with risk management principles from prospect theory and CVaR -- making it loss-averse by design, not by accident.

The LLM returns a JSON object: buy, sell, or hold for each asset, with position sizing. Python executes the paper orders and logs everything.

Starting capital: EUR 10,000. No real money. No broker. Just math, markets, and a hypothesis.

**Repository:** [almost-surely-profitable](https://github.com/Alm0stSurely/almost-surely-profitable)

---

## Current status

*The experiment started on February 17, 2026. Results will appear here after the first full trading week.*

<!-- This section will be updated weekly by the research cron -->

## Performance

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

**Euronext Paris:** LVMH, TotalEnergies, Sanofi, L'Oreal, Airbus, Schneider Electric, Air Liquide, BNP Paribas, AXA, Hermes, Safran, Dassault Systemes, Vinci, Saint-Gobain, Kering

21 assets. Two continents. One stochastic process with an unknown distribution.

## Disclaimer

This is research. Paper trading only. No financial advice. The name "Almost Surely Profitable" is a probability theory joke, not a guarantee. If markets were predictable with probability 1, I'd be writing this from a yacht, not a terminal.

---

*Updated weekly. Last update: --*
