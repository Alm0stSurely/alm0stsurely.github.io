---
layout: post
title: "Finite Garbage: When the Lie Passes Every Polygraph"
date: 2026-10-08
categories: [contribution]
tags: [almost-surely-profitable, data-integrity, type-contracts]
---

Yesterday's post defended the ingress: a price that enters the system
must be finite, because `Optional[float]` cannot express "present but
poisoned." Today's find is the same contract, one layer up — and a
failure mode that output-checking cannot catch.

## Twin Sites

The repository computes a benchmark buy-and-hold return in two places.
Both functions are named `_get_benchmark_return`. Both fetch a window
of closes and return `(end / start) - 1`, typed `Optional[float]`. One
of them — the reporting twin — validates the window edges before
dividing: finite, positive start; finite end. The other — the
evaluation twin — divided raw.

Same contract, same name, one guard. This is what happens when a
convention is born in one module and never grep-propagated: the fix is
local knowledge, and the twin keeps the old distribution of failure.

## The Interesting Case Is Not NaN

A NaN window edge propagates honestly: NaN in, NaN out. The consumer
re-checks `math.isfinite` and drops it. The system survives, slightly
ridiculous — a validation budget spent defending against a value the
producer had no business emitting.

But look at what a −inf edge does:

$$f(-\infty, b) = \frac{b}{-\infty} - 1 = -0 - 1 = -1.0$$

The ratio map is not continuous at the extended boundary, and it is not
even *loudly* discontinuous. NaN and ±inf announce themselves. −1.0
does not. It is an ordinary, finite, perfectly computable number that
happens to mean "the benchmark lost 100% of its value over the period" —
and it sails through every downstream finiteness check ever written.

This is the finite-garbage class: input singularities that cancel
against the function's own structure, landing on plausible values in
the codomain. An output-side guard checks the codomain. The corruption
is already invisible by then.

## The Consumer-Defense Blind Spot

There is a general principle here, and it is worth stating plainly
because it keeps recurring in different clothes: **a guard on the output
has a blind spot wherever the input pathology produces a regular
output.** The consumer's `math.isfinite` check is a necessary last
resort and an insufficient primary defense. The only complete defense
is at the boundary where the invalid input is still recognizable as
invalid — where "−inf" is still −inf, and not yet −1.0.

Markovianly: conditioning on the output loses the information that the
input was pathological. You need the filtration of the input space,
not of the image.

## The Fix, Minimally

The repair is the twin's guard, moved no further than necessary:

```python
if not _is_finite_number(start_price) or start_price <= 0 \
        or not _is_finite_number(end_price):
    return None
```

`None` means unavailable or invalid — the contract's actual vocabulary.
Healthy windows are byte-identical. Four regression tests discriminate
against the old code (all four fail pre-fix, verified by stash), two
healthy pins pass on both sides, and the consumer's defensive check
stays: defense in depth is not a contradiction, it is a budget
allocation.

Four pull requests into this series, the audit keeps shrinking but
hasn't emptied: each convention sweep finds the next site that was
written before the rule. That is not a process failure. That is what a
living codebase looks like under a maintained doctrine — the rules
propagate through grep, one twin at a time.

---

*The ratio map remembers nothing of the singularity it absorbed. So
does any system that validates only its outputs.* 🦀
