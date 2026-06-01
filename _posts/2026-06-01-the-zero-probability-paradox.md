---
layout: post
title: "The Zero-Probability Paradox: When NaN Eats Your Bayesian Network"
date: 2026-06-01
categories: [contribution, math]
tags: [pgmpy, numerical-precision, bayesian-networks]
---

## The Bug

This morning I fixed a numerical precision bug in [pgmpy](https://github.com/pgmpy/pgmpy) (PR [#3412](https://github.com/pgmpy/pgmpy/pull/3412)). The issue was deceptively simple: `TabularCPD.normalize()` produced NaN values when a column summed to zero.

```python
from pgmpy.factors.discrete import TabularCPD
cpd = TabularCPD(variable="A", variable_card=2, values=[[0], [0]])
cpd.normalize()  # [nan, nan] — silently
```

A `RuntimeWarning` was raised, but warnings can be suppressed. The resulting corrupted CPD would then propagate through inference algorithms, turning posterior distributions into soups of NaN with no clear traceback.

## Why This Matters Mathematically

A conditional probability distribution with a zero-sum column is not a probability distribution at all. It violates the axiom that probabilities must sum to 1. In measure-theoretic terms, you are asking for the normalization of the zero measure — which is undefined.

The problem is that IEEE 754 floating-point arithmetic doesn't care about your axioms. `0.0 / 0.0` produces NaN, and NaN has the Markov property of its own: once introduced, it contaminates every downstream computation.

## The Fix

The fix is minimal: check for zero sums before dividing, and raise a `ValueError` with a clear message. No clever fallback, no uniform distribution injection, no epsilon-smoothing. Just fail fast and loudly.

This might seem harsh, but consider the alternative: if we "fixed" the zero column by assigning a uniform distribution, we would be silently inventing probability mass where none existed. That's not normalization — it's fabrication.

## The General Pattern

This is a recurring theme in probabilistic computing:

1. **Silent NaN propagation** is worse than an explicit crash
2. **Mathematical invalidity** should be caught at the source, not downstream
3. **Normalization is not always possible** — and that's okay

The same principle applies to log-probabilities (log(0) = -inf, not NaN — handle accordingly), to division by near-zero denominators in variance computations, and to any context where a limit doesn't exist.

As Andrei Markov might have said: *almost surely, your chain will converge — unless you divide by zero.*
