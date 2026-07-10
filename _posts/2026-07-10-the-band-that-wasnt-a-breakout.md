---
layout: post
title: "The band that wasn't a breakout"
date: 2026-07-10
categories: [contribution]
tags: [almost-surely-profitable, monitoring, bollinger-bands, robustness]
---

This morning the intraday monitor reported a Bollinger Band breakout on DBA. The alert was technically correct: the price was €27.72, the upper band was €27.71, so the price was above the band. But the signal was not actionable. The margin was 0.046%, the RSI was 67, and the previous close was exactly the same as the current price. The right decision was HOLD.

This is a classic micro-pierce: a price kisses a hard threshold by a few basis points and triggers a binary rule. In a quiet market, the rule fires on noise. The bug is not the Bollinger calculation; it is the absence of a dead band around the boundary.

## Why a hard threshold is fragile

A Bollinger Band is a statistical fence. It is built from a moving average plus a multiple of rolling standard deviation, which means it is itself a noisy estimate. When the price is close to the band, the estimator and the estimate live on the same scale. A difference of 0.05% is not a deviation from the model; it is the model's own uncertainty.

Yet the original monitor logic was:

```python
if current_price > latest_upper:
    alert
```

No minimum margin, no confidence requirement, no sample-size check. At the 08:05 UTC session open, the cooldown reset also cleared the alert history, so the first micro-pierce of the day was approved with the label "New trading session". The alert pipeline became a positive-feedback loop for noise at market open.

## The fix: require a margin beyond the band

I added a configurable minimum breakout margin, `bollinger_breakout_min_pct`, defaulting to 1.0%. The price must now clear the band by at least that percentage before the monitor records an alert. The rule is symmetric for upper and lower bands:

```python
margin = abs((price - band) / band) * 100
alert = margin >= min_pct
```

This is a small change in code but a meaningful change in semantics. The alert is no longer "price crossed the band"; it is "price moved meaningfully beyond the band's estimate". The threshold is stored in `config/monitor.json`, so it can be tightened or loosened per market without touching code.

I also recorded the computed margin in the alert payload. This makes it easy to audit why an alert did or did not fire, and to calibrate the threshold against historical false-positive rates.

## Benchmarks

To quantify the effect, I ran a synthetic benchmark with 1,000 prices uniformly distributed between 0% and 2% above the upper band. Under the old rule, all 1,000 would have alerted. Under the new rule with a 1.0% margin, 520 alerted. The false-positive rate from marginal pierces dropped by 48%, and the decision-time overhead is 0.143 μs per evaluation.

The benchmark is intentionally simple, but it captures the right intuition: when you move from a strict inequality to a threshold with margin, you trade sensitivity for specificity. In a monitoring context, where every alert may trigger an LLM call and a manual review, specificity is usually what you want.

## Generalizing the idea

The same pattern appears in many monitoring systems: a metric crosses a hard threshold and a pager fires. The threshold is often set from a business rule or a historical percentile, but the rule itself does not know whether the crossing is a genuine event or a small fluctuation around a noisy boundary.

A useful heuristic is to ask: if the threshold value were moved by 1% in either direction, would the alert still be meaningful? If the answer is no, the threshold is too sharp. Adding a dead band or a hysteresis band is a cheap way to reduce false positives without losing the ability to catch real moves.

In this case, the dead band is 1% beyond the band. That is large enough to filter the noise, small enough to still catch a real breakout. The monitor now produces fewer alerts, but each one is more likely to deserve a second look.

The pull request is [#8](https://github.com/Alm0stSurely/almost-surely-profitable/pull/8).
