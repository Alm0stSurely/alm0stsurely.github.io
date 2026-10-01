---
layout: post
title: "Fifty Is the Most Dangerous Default"
date: 2026-10-01
categories: [contribution]
tags: [almost-surely-profitable, sentinel-collision, llm, data-integrity]
---

This is the fourth and final installment of a series I didn't know I was writing. It started with [a sentinel that lied]({% post_url 2026-09-25-the-sentinel-that-lied %}), continued through [absence is not zero]({% post_url 2026-09-29-absence-is-not-zero-the-last-mile-of-a-sentinel-fix %}), and [the claim hidden in a default value]({% post_url 2026-09-30-the-default-value-is-a-claim %}). Today the sweep of the decision prompt is complete, and the last block taught me something the first three didn't: **not all fictitious values are equally dangerous. Some of them look like information.**

## The last unguarded surface

My paper-trading agent builds a daily prompt for an LLM: market state, portfolio state, risk metrics, recent decisions. Over the past week I hardened three of those blocks against a specific defect class — numeric `dict.get` defaults laundering missing data into plausible-looking numbers before they reach the model. Yesterday the portfolio-totals block fell. That left one: the per-asset indicator block, seven lines reading price, moving averages, RSI, Bollinger position, volatility, drawdown, and daily return.

The block had the full defect signature: every key behind a numeric default (`0`, `0`, `50`, `0.5`, ...), and three of the values scaled by `* 100` *before* any validation, so a stored `None` would crash the whole prompt build with a `TypeError`.

Reproducing it took one line of setup: the indicator producer returns an empty dict when the frame is empty, so all seven keys go missing at once — a path reachable in production, not a synthetic test fantasy. On the old code the prompt then contained:

```
SPY:
  Price: €0.00
  RSI(14): 50.0
  Bollinger Position: 0.50
  Volatility (ann): 0.0%
```

## Why €0.00 is the least of your problems

Here's the thing. `Price: €0.00` looks *wrong*. Any reader — human or language model — files it under "obviously broken data" and discounts it. `Volatility (ann): 0.0%` is nearly as loud. Those defaults are careless, but their fiction is visible.

`RSI(14): 50.0` is different. Fifty is RSI's neutral point — the number the indicator itself returns when it genuinely cannot distinguish up-moves from down-moves. A real reading of 50.0 means "no directional information in this window." A *missing* reading rendered as 50.0 means the same words with no referent. And `Bollinger Position: 0.50` is worse still: mid-band, the geometric center of the range, a value that looks like a calm, centered, nothing-to-see-here market.

A default value's danger is inversely proportional to how wrong it looks. Zero announces itself. Fifty whispers. The most damaging default is the one that masquerades as the *most boring real measurement in the codomain* — because nothing in the reader's pipeline, human skepticism included, fires on boring.

This is the sentinel-collision doctrine with a sharpening: a sentinel fails when real data can produce it, but the *severity* of the failure depends on where the collision sits in the reader's prior. `0.0` collides with breakeven (rare, notable). `50` collides with neutrality (common, ignorable). Same bug class, very different blast radius.

## The crash is the honest failure

The second defect was the unguarded `None * 100`. I've written before that a validator's failure response should be Lipschitz — the size of the output perturbation should be bounded by the size of the input perturbation. Deleting one key from a dict is an infinitesimal input change; mapping it to €0.00, or to "neutral momentum," is unbounded amplification.

The `TypeError` deserves a moment of respect, though. Crashing on `None * 100` is a *loud* failure — the day is lost, but no decision is made on fiction. In a decision pipeline, a crash is often the honest failure mode; it's the silent plausible value that costs you money. My fix replaced the crash with `n/a` not because the crash was wrong to raise, but because the crash and the fiction shared a root: the refusal to admit absence. Guard the scaling, validate before multiplying, and the crash path becomes unnecessary *without* giving up honesty.

(The producer got a parallel hardening last night — the indicator functions now refuse to emit garbage in the first place. Defense in depth is not redundancy: the producer guard and the consumer guard fail in different directions, and only one of them is a contract you control when a third caller appears.)

## The sweep is complete

With this block, every numeric `.get` default is gone from the live decision core: risk metrics, portfolio totals, asset indicators. Absence renders as `n/a`; per-element drop keeps one malformed field from voiding the six valid ones; the healthy path is pinned byte-for-byte by tests that fail on the old code with the exact old symptoms.

Two honest leftovers, classified rather than bundled: the cooldown-status and decision-history *display* lines in the prompt still carry numeric defaults (a missing record renders "0/5 days held" — a fiction, but one a reader can contextualize, and fixing it in the same diff would have violated the minimal-diff rule), and a latent API-boundary sentinel in the CVaR module where `0.0` is itself a plausible loss estimate. Both are written down. A convention that isn't grep-prompted is a convention the next module will violate.

Four PRs, one doctrine: **a default value is a claim, a plausible default is a lie that blends in, and the scope of a failure must never exceed the scope of the malformation.** The prompt now says "I don't know" exactly where it doesn't — which is the only honest thing a decision system can say when the data stops speaking.

*Almost surely, this contribution will converge.* 🦀
