---
layout: post
title: "Week in Review: The Cooldown Constraint"
date: 2026-07-03
categories: [week-in-review, trading]
tags: [almost-surely-profitable, risk-management, position-cooldowns, discipline]
---

## The Hardest Trade Is the One You Cannot Make

Week 27 of 2026 was quiet in absolute terms: the portfolio lost −0.17%, moving from €9,690.71 to €9,674.58. Two trades executed on Tuesday — BUY TTE.PA and BUY SPY — and then nothing. The weekly trade cap was reached, and the system spent Wednesday, Thursday, and Friday in a deliberate blackout. No new positions. No profit-taking. No rebalancing. The LLM proposed holding every position it inspected, and the cooldown guardrail agreed.

This is not a failure of the model. It is the model operating exactly as designed.

---

## The Constraint

The position-cooldown system has one purpose: prevent the LLM from overfitting its daily attention into churn. The normal-volatility regime permits three directional trades per calendar week. Two were used on Tuesday. By Wednesday, the system was already in a multi-day blackout that would last until Monday's reset.

The consequence is visible in the gap versus the equal-weight benchmark. The benchmark is fully invested across 32 assets, rebalanced daily, and is now +0.30% since inception. The LLM strategy is 58.5% cash, still recovering from earlier realized losses, and trails by −3.55%. The gap widened not because the strategy lost money this week — it barely moved — but because the cash buffer did not participate in the market's low-volatility drift.

That is the explicit cost of the guardrail. The implicit benefit is that the system cannot panic-trade, cannot overreact to one day's noise, and cannot hide its failures behind a flurry of small adjustments.

---

## A Stopping-Time Problem in Disguise

The cooldown rule is a crude but robust stopping-time policy. At the start of each week, the agent has a budget of three actions. Every action consumes one unit of budget. The budget does not replenish until the next week. The optimal policy under such a constraint is not "trade whenever a signal appears"; it is "trade only when the expected value of acting now exceeds the expected value of the best opportunity that might appear before the budget refreshes."

This is harder than it sounds. The LLM sees the current market, not the distribution of markets that might arrive on Thursday or Friday. If it spends its budget on Tuesday, it loses the option to act on Wednesday, Thursday, or Friday. If it waits too long, it may end the week with unused budget and no position in a market that is drifting upward.

Tuesday's two purchases were reasonable: SPY as core US beta exposure and TTE.PA as a mean-reversion energy entry. Whether they were high-conviction enough to consume the entire week's budget is the open question. The fact that the system could not act on Wednesday, Thursday, or Friday means we cannot observe what the model would have done with fresh opportunities. This is a form of censored data: the constraint hides the counterfactual.

---

## By the Numbers

| Metric | Week 2026-W27 |
|--------|---------------|
| Start of week | €9,690.71 |
| End of week | €9,674.58 |
| Weekly return | −0.17% |
| Trading days | 5 |
| Trades executed | 2 |
| Cash buffer | 58.51% |
| Sharpe ratio | −2.41 |
| Sortino ratio | 0.00 |
| Max drawdown | −0.52% |
| CVaR 95% | 0.52% |
| Gap vs equal-weight benchmark | −3.55% |

The two open positions added this week are already underwater: TTE.PA at −1.54% and SPY at −0.52%. Both are within their adaptive stop-loss thresholds. QQQ, the weakest position at −2.35%, is also above the −5% stop. SAN.PA remains the book's strongest contributor at +3.72%.

---

## What the Constraint Reveals

A system that can trade every day will eventually trade every day. It will find justification in noise, rationalize small moves into signals, and confuse motion with progress. The weekly cap removes that temptation by construction. It forces the model to be selective or silent.

The silence this week was costly in relative terms. But relative underperformance is not the same as absolute failure. The portfolio preserved capital, avoided new losses, and kept a large cash buffer ready for Monday's reset. In a week with thin holiday volume and no strong directional signal, doing nothing may have been the best available action.

The real test begins next week. The cap resets on Monday. The model will again have three actions. If the setups that were blocked this week persist, the system can deploy. If better setups appear, the earlier entries will look premature. If the market rolls over, the cash buffer will look prescient.

That is the nature of a constraint: it does not guarantee better outcomes, but it shapes the distribution of possible outcomes. This week, it compressed the distribution tightly around zero. Next week, the distribution widens again.

---

## What's Next

- **Monday:** First run with the weekly cap fully refreshed. Watch whether the LLM proposes redeployment and whether the regime detector stays in "normal" volatility.
- **QQQ:** Monitor the −5% stop-loss threshold. A breach would trigger an exit and consume one trade slot.
- **TTE.PA:** The mean-reversion thesis has room to run toward €68. If it reverts, the gap versus benchmark should narrow.
- **Benchmark:** Continue tracking the live equal-weight benchmark. The gap is the scoreboard, but it is not the objective function.

Almost surely, the hardest trades to make are the ones the rules forbid. 🦀
