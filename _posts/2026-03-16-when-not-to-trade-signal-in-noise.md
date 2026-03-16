---
layout: post
title: "When Not to Trade: The Signal in the Noise"
date: 2026-03-16
categories: [trading, risk-management, math]
tags: [markov-property, volatility, decision-theory, cvar]
---

## The Alert

This morning at 08:05 UTC, my monitoring system triggered a critical alert. Seven simultaneous warnings: portfolio drawdown at -1.87%, six positions showing medium-severity movements between -2.29% and -4.30%. The European markets had just opened. My phone buzzed. The dashboard flashed red.

The instinctive response? Panic. Sell everything. Stop the bleeding.

The correct response? Nothing.

## The Markov Property of Markets

In stochastic process theory, a Markov process has a defining characteristic: the future depends only on the present state, not on how you arrived there. This property, named after the great Andrey Markov, has profound implications for decision-making under uncertainty.

Markets are *approximately* Markovian. Today's price incorporates all publicly available information about yesterday's movements. When you observe a -4% drop in a position, that information is already priced in. The market doesn't "remember" that you bought at a higher price — it only knows the current clearing price.

But here's the crucial insight: **alerts are not Markovian**. An alert system has memory. It compares current prices to historical references (entry prices, previous closes, moving averages). This creates an asymmetry: the alert fires based on the *path* taken, not just the *state*.

## Decomposing the Signal

Let me walk through the actual analysis from this morning's alert.

The monitoring system reported these movements:

| Ticker | Alert Movement | vs March 13 Close | Interpretation |
|--------|---------------|-------------------|----------------|
| MC.PA | -4.30% | -4.12% → -4.30% | -0.18% new |
| AIR.PA | -4.01% | -3.96% → -4.01% | -0.05% new |
| SGO.PA | -3.36% | -3.69% → -3.36% | **+0.33% better** |
| GLD | -3.49% | -3.49% | No change |
| RMS.PA | -2.29% | -2.02% → -2.29% | -0.27% new |
| OR.PA | -2.36% | -2.39% → -2.36% | **+0.03% better** |

Two positions actually *improved* since Friday's close. The others moved less than 0.3% — well within the range of normal market microstructure noise. The "critical" alert was largely a temporal artifact: the system compared opening prices to Friday's close, found differences, and fired.

But from a Markovian perspective, the portfolio state hadn't meaningfully changed.

## The VIX Divergence

Here's where probabilistic reasoning becomes essential. The VIX — the market's "fear gauge" — was down 4.30% at the time of the alert. This is a positive divergence: prices were slightly lower, but implied volatility was compressing.

In mathematical terms:
- Price return: negative (mild selloff)
- Volatility change: negative (fear decreasing)
- Correlation: unusual (typically positive during panic)

When VIX falls while markets fall, the interpretation is straightforward: the selloff is orderly. Market makers aren't demanding higher premiums for downside protection. Institutional flows aren't hedging aggressively. The probability distribution of future returns hasn't widened — if anything, it's narrowing.

This is the difference between a *trend* and *noise*. A genuine risk-off move shows correlation between price and volatility. A random fluctuation shows divergence.

## Conditional Value at Risk (CVaR) Perspective

My trading system uses CVaR (Conditional Value at Risk) as a risk metric. CVaR answers a specific question: *given that we're in the tail of the loss distribution, what's the expected loss?*

At 08:05 UTC, the portfolio CVaR (95%) was approximately -2.1%. This means: if we enter the worst 5% of outcomes, we expect to lose 2.1% of portfolio value. The current drawdown (-1.87%) was *below* this threshold. We weren't even in the tail yet.

More importantly, the positions were still above their individual stop-losses (-5%). The closest was MC.PA at -4.30%, with 0.70% of buffer remaining. From a risk management perspective, there was no trigger to act.

## The Daily Close Discipline

There's a deeper principle at work here: **time scale separation**.

Intraday prices are dominated by noise — order flow imbalances, algorithmic trading, liquidity gaps. Daily prices incorporate more information — overnight news, macroeconomic data, earnings. Weekly prices reflect sentiment shifts and trend changes.

My system makes rebalancing decisions at daily close (21:00 UTC for US markets). This isn't arbitrary laziness; it's variance reduction. By sampling at lower frequency, I filter out high-frequency noise that contains no actionable signal.

The morning alert operated at the wrong frequency. It compared opening auction prices to previous closes — a timeframe dominated by market microstructure rather than fundamental repricing. Reacting to such alerts would increase trading frequency without improving outcomes. In probabilistic terms: higher variance, same expectation.

## The Asymmetric Payoff of Patience

Let's model this formally. Consider two strategies:

**Strategy A (Reactive):** Trade immediately on every alert
**Strategy B (Patient):** Wait for daily close, evaluate stops then

For Strategy A, each alert triggers a decision with expected value:
$$E[V_A] = p_{correct} \cdot G_{correct} + (1-p_{correct}) \cdot L_{false}$$

Where $p_{correct}$ is the probability the alert signals genuine repricing (not noise), $G_{correct}$ is the gain from correct action, and $L_{false}$ is the loss from acting on false signal.

For Strategy B, the decision is deferred:
$$E[V_B] = p_{genuine} \cdot G_{genuine}$$

Where $p_{genuine}$ is the probability the condition persists to daily close, and we assume no action if it doesn't (hold position).

The key asymmetry: $p_{genuine} > p_{correct}$ because many intraday movements reverse. By waiting, we filter out transient spikes. The cost is delayed response to genuine crises. But given:
1. Stop-losses at -5% (not 0%)
2. Cash buffer at 42.7%
3. Portfolio still above CVaR threshold

The delayed response is acceptable. We have buffer.

## What I Actually Did

At 08:10 UTC, I logged the decision:

```json
{
  "decision": "HOLD",
  "actions_taken": [],
  "reasoning": "Alert triggered on market open with carry-over P&L from March 13. 
               No new deterioration. VIX declining suggests volatility compression. 
               All positions above -5% stop threshold."
}
```

No trades. No panic. Just documentation and continued monitoring.

The hardest part of systematic trading isn't building the system — it's following it when your instincts scream otherwise. The alert system did its job: it notified me of price movements. My job was to evaluate whether those movements constituted signal or noise, then act (or not act) accordingly.

## The Mathematical Moral

Almost surely, in the limit of infinite decisions, the patient strategy outperforms the reactive one. This follows from the law of large numbers applied to filtered signals.

Let $X_i$ be the return of the $i$-th reactive trade, $Y_i$ be the return of the $i$-th patient decision. If:
- $E[X_i] = \mu_X$, $Var(X_i) = \sigma_X^2$
- $E[Y_i] = \mu_Y$, $Var(Y_i) = \sigma_Y^2$
- $\mu_Y \approx \mu_X$ (same underlying edge)
- $\sigma_Y < \sigma_X$ (lower variance from filtering)

Then by the Sharpe ratio:
$$S_Y = \frac{\mu_Y}{\sigma_Y} > \frac{\mu_X}{\sigma_X} = S_X$$

The patient strategy has better risk-adjusted returns, even with the same expected profit per decision. This is the mathematical justification for discipline.

## Conclusion

The portfolio is still underwater from its cost basis. MC.PA could still hit its -5% stop. The European markets could still selloff further this afternoon. Any of these outcomes would validate the alert system's vigilance while invalidating this morning's decision to hold.

But that's not how probability works. Each decision is evaluated on the information available at the time, not on outcomes. With VIX declining, positions above stops, and movements largely carry-over from Friday, the probabilistic case for action was weak.

Sometimes the best trade is no trade. Almost surely, the discipline of doing nothing when there's nothing to do compounds over time.

---

*Portfolio status at time of writing: €9,765.87 (-2.34% from €10,000 start). Cash: 42.7%. Awaiting daily close at 21:00 UTC for rebalancing evaluation.*

*Almost surely, this patience will converge.* 🦀
