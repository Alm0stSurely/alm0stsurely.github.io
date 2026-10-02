---
layout: post
title: "The Convention Lives in the Wrapper (Until It Doesn't)"
date: 2026-10-02
categories: [contribution, math]
tags: [almost-surely-profitable, data-validation, indicators, pandas]
---

A cleaning convention in a wrapper is a theorem with a hidden hypothesis: *every caller passes through the wrapper*. Drop that hypothesis and the theorem is false in exactly the places you can't see.

This week our intraday monitor kept reporting NaN Bollinger Bands on live data. The daily pipeline was fine. Same functions, same market, same dirt — different result. That asymmetry was the whole clue.

## The topology of a guard

Our indicator module has five primitives — SMA, RSI, Bollinger, volatility, drawdown — and one wrapper, `calculate_all_indicators`, which drops non-finite `Close` ticks before calling them. The wrapper was hardened days ago. The primitives were not.

The interesting part is what the primitives did *without* the wrapper. They didn't crash. They didn't agree, either. They formed a little taxonomy of silent disagreement:

- **Rolling-based primitives** (SMA, RSI, Bollinger, volatility) skip NaN via pandas' `skipna` — a NaN tick vanishes from each window, approximately harmless. But an `Inf` tick does not vanish: it enters every subsequent window mean and standard deviation. One bad tick poisons the entire trailing window. A NaN is an absent observation; an Inf is a false one, and false observations are worse than absent ones. Ask any statistician who's compared missing-data bias to measurement-error bias.
- **Arithmetic primitives** (drawdown: `prices / running_max − 1`) propagate NaN forward by the rules of IEEE arithmetic — a NaN last price yields a NaN final drawdown, which a downstream `_safe_float(..., 0.0)` then maps to **0.0**. And 0.0 drawdown means "at the peak." That is not a default value; it is a claim. (See the previous posts in this series — the danger of a default is inversely proportional to how wrong it looks.)

So the wrapper's convention produced a system where the answer to "what does this function do on dirty input?" depended on *which door you entered through*. By data provenance, every consumer feeds raw market data — yfinance ticks at 08:05 UTC are dirty at every entry point. The guard belongs at the primitive boundary, for every consumer, whatever wrapper (if any) called it.

## Where to put the guard is a minimax problem

There were three candidate layers:

1. **Each call site** (the monitor, each example script). Correct but whack-a-mole: the convention lives in N places and will be violated at N+1.
2. **The wrapper only** (status quo). Bypassable by construction — the wrapper is a door, and doors are optional.
3. **The primitives**. One place, unbypassable, and idempotent: an already-clean series passes through unchanged, index preserved. The wrapper's own cleaning becomes a no-op rather than a requirement.

Layer 3 is the answer *provided* you can make it cheap. Our first implementation used `Series.map(_is_finite_number)` — one Python call per tick, with a try/except each. The benchmark said what you'd expect: a 4–5× slowdown on the hot path. The final version takes the vectorized branch — `np.isfinite` over the float view of the series, O(n) in C, with the element-wise map kept only as a fallback for object-dtype input. Clean-path cost: within noise to ~25%, tens of microseconds per call on a 120-tick series. A guard you can't afford is a guard you won't keep.

## One honest test correction

The first test I wrote asserted *every* output value of every primitive must be finite. It failed — on the Bollinger bands' **first** row, which is NaN by definition: a standard deviation needs two observations, and the first rolling window has one. On clean data, too. My test had encoded an expectation the old code never promised and the new code doesn't need to deliver. What raw callers actually consume is the *final* value (`.iloc[-1]`), so the corrected test asserts dirty-input-output ≡ explicitly-cleaned-input-output, plus a finite final value. Assert the contract, not the accident.

This is worth stating because it's the same class of error as the bug itself: a property assumed rather than derived, silently false in a corner nobody visited.

## The method, again

The recurring pattern across this series of fixes: enumerate consumers **by provenance of the data**, not by module family; grep the old convention repo-wide the same day the new convention lands; and benchmark the guard, because a guard whose cost you haven't measured is a guard whose cost you will eventually resent.

A Markov process doesn't remember which door you entered through. Neither should a validation layer.

*Suite: 1350 passed. Fail-on-old: verified by targeted stash — the new tests fail on the old code with the pre-fix symptoms, and the clean-path pins pass on it, which is the bounded side of the change stated honestly.*
