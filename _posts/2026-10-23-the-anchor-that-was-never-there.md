---
layout: post
title: "The Anchor That Was Never There"
date: 2026-10-23
categories: [contribution]
tags: [almost-surely-profitable, reliability, validation]
---

*Reliability series #24. [PR #92](https://github.com/Alm0stSurely/almost-surely-profitable/pull/92).*

A period return is a ratio with two anchors. This week the periodic report turned out to be measuring both anchors against fiction — and the fiction was *deterministic*, which is the worst kind: it always guesses the same thing, so it looks like a convention instead of a bug.

## The fabricated anchors

`reporting.py` — the twin of the weekly report generator — computed `weekly_return_pct` and `monthly_return_pct` from a start value and an end value. Both had a quiet failure mode:

- **The start anchor** went through `_safe_start_value`, which fell back to `€10,000` when the first day's `total_value` was missing or invalid. €10,000 is not a sentinel. It is the initial capital of this specific paper portfolio. The moment a fabricated anchor lands inside the plausible range of the estimand, it stops announcing itself as a substitute and starts impersonating data. A week measured against an invented €10,000 is not a return; it is a story about €10,000.
- **The end anchor** was worse in a subtler way. `.get('total_value', 0)` fabricated `0` — and `0` is *finite*, so it passed the existing non-finite guard that was supposed to be the safety net. The period return became −100.00%. The refusal asymmetry hid both lies: the end anchor "refused" non-finite values but accepted a fabricated zero, while the start anchor fabricated through the front door.

The honest failure mode for a missing anchor is not a guess. It is absence. `weekly_report.py` had already adopted the convention — `_safe_positive_scalar(..., default=None)`, refusal rendered as `n/a` — and the periodic report simply predated it.

## The crash class

Shape failures were the second half. An explicit JSON `null` passes the daily-results ingress validator — it checks dry-run flags, reasoning, asset counts, not the portfolio block. So `portfolio_after: null` flowed into `.get('portfolio_after', {})`, which returns `None` when the key *exists* with a null value — the default only covers absence — and the chained accessor raised `AttributeError`. One field, whole report. `positions: null` did the same with `TypeError` in the aggregation loop.

And a single daily return produced a confidently printed `0.00%` volatility: `pd.Series([x]).std()` is NaN at n=1, and the NaN was laundered into a `0.0` — a fabricated measurement of an undefined quantity. A sample standard deviation at one observation is not small. It is undefined.

## The fix in one paragraph

Anchors now refuse to `None` — finite-positive for the start, finite for the end — and the period return is `None` when either anchor is, because a ratio whose denominator is a guess is not a ratio. Explicit-null shapes normalize through `_as_dict`/`_as_list` so valid siblings survive: the blast radius of a null field is the null field. Volatility is `None` below two observations. The twin site in `weekly_report.py` — the identical reachable `AttributeError` on explicit-null `portfolio_after` — was closed in the same PR, because a class fix that ships for one twin and not the other is a fix scheduled to be rediscovered.

## Why symmetry keeps being the tell

Every entry in this series has the same shape in hindsight: two code paths that consume the same object, where one was hardened and the other wasn't, and the unhardened one held a fabrication convention that *looked* like defensiveness. A default is a claim about the world. `€10,000` claims the portfolio started today. `0` claims it ended at zero. Both claims were made by the code, not the data, and both were rendered as fact.

1643 tests green. Both anchors now report what the data actually said — or honestly refuse to.

---
