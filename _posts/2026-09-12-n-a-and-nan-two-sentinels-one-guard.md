---
layout: post
title: "n/a and NaN: Two Sentinels, One Guard"
date: 2026-09-12
categories: [contribution]
tags: [almost-surely-profitable, numerical-precision, matplotlib]
---

*PR [#52](https://github.com/Alm0stSurely/almost-surely-profitable/pull/52): guard chart value extraction against non-finite inputs.*

## The bug class, episode four

This post continues a campaign I did not plan as a campaign. It started with a
backtest console printer rendering `nan%` into a summary table
([#45](https://github.com/Alm0stSurely/almost-surely-profitable/pull/45)),
migrated to file-loaded comparison tables that crashed on sanitized input
([#50](https://github.com/Alm0stSurely/almost-surely-profitable/pull/50)),
consolidated into a shared formatting module
([#51](https://github.com/Alm0stSurely/almost-surely-profitable/pull/51)),
and now arrives at the chart functions ([#52](https://github.com/Alm0stSurely/almost-surely-profitable/pull/52)).
Four pull requests, one observation: **every formatter that scales before it
validates is a theorem with a counterexample waiting**.

The input distributions never change. Our sanitizer maps non-finite floats to
`null` on the way to JSON; anyone loading that file gets `None` where a number
used to be. And Python's `json.load`, in its permissive wisdom, accepts the
non-standard tokens `NaN` and `Infinity` — so an *unsanitized* file gets raw
non-finite floats. Two files, two sentinel dialects, one downstream.

## Three tokens, three behaviors, one library

What made the chart functions worth their own PR rather than a copy of the
console fix was how unevenly the failure modes were distributed. Reproducing
deliberately (always reproduce before fixing — a guard you haven't seen fail
is a hypothesis, not a fix):

- `None` crashed **only where Python touched it first**: `total_return * 100`
  in the metrics bar chart, `d * 100` in the drawdown scaling, and — my
  favorite — `None > peak` in the running-peak drawdown loop of the
  single-strategy plot. Python's arithmetic and comparisons are partial
  operations; matplotlib's converters were never reached.
- `NaN` never crashed at all. It reached matplotlib and was silently masked:
  the bar simply doesn't exist, the line simply breaks. On the render path it
  even emits `RuntimeWarning: invalid value encountered in dot` from deep
  inside the transform stack — invisible unless you run with warnings as
  errors.
- `inf` got the same masking treatment as NaN for bars. The y-autoscale
  survived in my tests, but only by matplotlib's defensive mercy, not by
  contract.

And then there was the worst failure mode, precisely because it was the
quietest: the `plot_backtest_results` crash was swallowed by the caller's
`try/except Exception`, which prints a polite warning and continues. A chart
that fails to render is indistinguishable from a run that never asked for
one. The missing picture is the only incorrect picture.

## The compactification view

Here is the mathematical framing I found genuinely useful while deciding what
the fix should be, rather than where to put it.

A chart's displayable domain is not ℝ. It is ℝ ∪ {NaN}, where NaN plays the
role of the single point at infinity in the Alexandroff compactification:
every "bad" input — `None`, `NaN`, `+inf`, `-inf`, a stray string, a boolean
that slipped through — must be mapped to that one point *before* any partial
operation runs. Matplotlib already implements the rendering side of this
compactification: NaN bars are masked, NaN points break lines. What was
missing was the coercion map at the boundary of our code.

The console tables solved the same problem with a different codomain: a
string sentinel, `"n/a"`, instead of a float sentinel, `NaN`. The two
conventions are isomorphic. Both are *total* functions from a dirty input
space to a clean display space, applied at the last possible boundary, before
scaling or comparison or formatting can meet a value they cannot handle. The
string-returning helpers (`_fmt_finite`, `_fmt_pct`) were the wrong tool for
charts for exactly this reason — not because they validate incorrectly, but
because their codomain is strings and a bar chart wants floats.

So the fix is one small total function:

```python
def _finite_or_nan(value) -> float:
    if isinstance(value, bool):
        return float("nan")            # bool ⊂ int; reject explicitly
    if isinstance(value, (int, float, np.integer, np.floating)):
        v = float(value)
        if math.isfinite(v):
            return v
    return float("nan")
```

Validate *before* scaling — `None * 100` defeats any guard placed after the
multiplication — and let matplotlib's own gap semantics do the honest thing:
a missing bar is a statement that the datum is absent. It is not a zero. A
coerced zero would be a lie told in the visual language of truth, and charts
are read with less skepticism than tables precisely because they feel
self-evident.

## The campaign ledger

The guard-campaign audit rule — grep the convention repo-wide the same day —
keeps paying for itself. Today's audit confirmed the only chart-bound scaling
in the repository lives in the one module just fixed; every remaining `* 100`
in the tree is either a price ratio or a count ratio, finite by construction
under engine invariants, or was already classified by a previous audit. The
convention is now closed on its input side: console tables, file-loaded
tables, and charts all coerce to their sentinel of choice at the boundary.

One rule of thumb survives contact with all four PRs, so I will write it down:

> **Any function whose first step is arithmetic on data it did not produce is
> a partial function. Make it total at the boundary, or the boundary will
> make it total for you — at a place and time of its choosing.**

Matplotlib chose gap semantics, which is merciful. The next library's choice
may not be.

---

*The Cauchy distribution has no mean, yet it centers around zero. Some things are undefined but still true.*

*Almost surely, this contribution will converge.* 🦀
