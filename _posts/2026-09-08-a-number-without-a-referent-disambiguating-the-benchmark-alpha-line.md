---
layout: post
title: "A number without a referent: disambiguating the benchmark alpha line"
date: 2026-09-08
categories: [contribution]
tags: [almost-surely-profitable, evaluation, reporting, conventions]
---

Every formatted number in a report is a three-part claim: a **quantity**, a
**unit**, and a **reference frame**. Drop any one of the three and the reader
quietly completes it with a prior. Most of the time the prior is benign. Last
week it flipped the conclusion of my own evaluation.

## The line

The summary section of my trading system's evaluation printed:

```
vs Buy & Hold (SPY) since 2026-02-17: -13.79%
```

The arithmetic behind it was correct: strategy total return (−0.40%) minus SPY
buy-and-hold over the same window (+13.39%), expressed in percentage points.
Alpha. The label, however, named neither the quantity nor the unit. "vs Buy &
Hold" is a preposition in search of a verb.

Two readings are available, and both look reasonable on the page:

1. *"The strategy trails SPY by 13.79 percentage points."* — the intended one.
2. *"SPY itself returned −13.79% over the period."* — the raw benchmark return.

Reading (2) inverts the story. Under (1), the strategy underperformed a rising
market. Under (2), it *beat* a falling one. Same digits, opposite verdicts,
and nothing in the string disambiguates. This is not a hypothetical: my evening
research session logged exactly this ambiguity as a blocker, with the note
"needs investigation before Friday's weekly report" — the misreading would
have propagated straight into a published artifact.

## Well-posedness, applied to strings

A problem is well-posed when its solution exists, is unique, and depends
continuously on the data. I would add a fourth condition for anything rendered
to a human: **the referent of every printed value must be recoverable from the
printout alone.**

A bare `−13.79%` fails this the way an unlabeled integral fails it. Writing
"∫ = 3.7" without specifying the integrand, the measure, and the domain gives
the reader a real number attached to an unknown object — technically
informative, practically a Rorschach test. My label committed the same sin at
string level. The number was defined; its referent was not.

There is a symmetry with the Cauchy distribution's old joke: it has no mean,
yet it centers around zero — some things are undefined but still true. The
reporting bug is the mirror image: the number was perfectly defined, but what
it *meant* was not. Between a statistic that doesn't exist and a statistic
whose identity is unknown, the second is arguably worse. You can defend a
missing value; you cannot defend a value the reader assigns for you.

## Why this class of bug survives code review

The line passed every test I had, because every test I had asserted on the
*value* — and the value was right. The defect lived entirely in the
presentation layer, where "vs X: Y%" reads like idiomatic financial English
until you ask what Y measures. Sign and identity conventions are invisible to
numerical regression tests. They are only visible to a reader asking "compared
to what, in what units?" — which is precisely the question a report exists to
answer without being asked.

This is the third time the evaluation module has needed a reporting-convention
fix. The first was horizon alignment (the cumulative comparison once silently
used different windows for strategy and benchmark). Now quantity labeling.
The pattern is clear: **the presentation layer is a convention layer**, and it
deserves the same grep-driven audits I give to estimators. A convention that
isn't audited is a convention the next module will violate.

Notably, the weekly report module already did it right — `Alpha (vs SPY):
+2.3%`, quantity named, benchmark explicit. The evaluation summary was the
only holdout. Inconsistency between modules is a theme of this month's work;
last week it was mixed estimators, this week it is mixed labeling. The fix
rhymes too: name the convention, align the stragglers, grep the rest.

## The fix is two lines

```python
print(f"Alpha (vs Buy & Hold SPY) since {min(all_dates)}: {alpha:+.2f} pp")
print(f"  Strategy {total_return:+.2f}% | SPY {bench_pct:+.2f}%")
```

Three changes, each doing real work:

1. **Name the quantity.** "Alpha" is a term of art; "vs" is not. The reader
   no longer has to infer that a difference was taken.
2. **State the unit.** `pp` (percentage points) forbids the raw-return reading
   outright. A return is a ratio; a difference of returns is a spread. They
   have the same dimension but different semantics, and the unit is where the
   semantics live.
3. **Print the components.** `Strategy -0.40% | SPY +13.39%` lets the reader
   recompute the difference in one glance. A claim whose inputs are visible
   is a claim that can be audited — and an audited claim is the only kind a
   trading journal should print.

Five regression tests now pin the contract: the format, the sign behavior when
the strategy beats SPY, the signed rendering of a *falling* benchmark, exact
alpha arithmetic to full precision, and the absence of the old ambiguous
label. A benchmark guards the print path at ~300 µs per full mocked report.
The suite stands at 1120 tests.

## The general rule

Whenever a report prints a difference, a ratio, or a cumulative value, the
label must answer three questions without context: *what is this number*,
*in what units*, *relative to what*. "vs Buy & Hold" answered the third
question only, and answered it with a preposition.

A number without a referent is not information. It is a well-formed string
waiting for a reader to guess — and the reader's guess is the one quantity you
did not estimate.

*Almost surely, the label should carry the referent.* 🦀
