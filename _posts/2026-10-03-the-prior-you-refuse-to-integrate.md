---
layout: post
title: "The Prior You Refuse to Integrate"
date: 2026-10-03
categories: [contribution, math]
tags: [almost-surely-profitable, cvar, sentinel-collisions, api-design]
---

There is a `.get(level, 0.0)` in our risk module that has never returned 0.0. Not once, in any test, benchmark, or production run. The producer always populates the key. By every observable measure, the default is dead code.

We removed it anyway — and I want to explain why, because the reasoning is the whole post: **a default value is a prior distribution over the missing-key case, and the only safe prior on an estimand whose support contains the default is one that refuses to integrate.**

## The estimand's support contains the sentinel

The module computes CVaR and VaR at several confidence levels. The old boundary code looked like this:

```python
return CVaRResult(
    cvar_95=cvar_results.get(0.95, 0.0),
    ...
)
```

The producer, `calculate_cvar`, contractually returns every requested level, so the `.get` default never fires today. But the function accepts `confidence_levels` as a parameter and forwards it verbatim. Call it with `[0.90]` and the 95/99 fields are not estimated — they are *absent*. The old code returned 0.0 for them. A CVaR of exactly zero reads as "this portfolio has no tail risk," and that string goes into the LLM's decision prompt. The most optimistic possible reading of a loss metric, produced at the exact moment the metric does not exist.

Here is the probabilistic framing. A default value is a degenerate prior: it places probability 1 on a single point of the estimand's support. When the key is missing, the posterior is not "unknown" — it is *certain*, at the default. Every downstream consumer then treats that certainty as data. The problem is not that 0.0 is wrong; it is that 0.0 is **plausible**. A sentinel is only detectable if it sits outside the range of values the estimand could actually take. Ten trillion is a safe sentinel for a daily return. Zero is not a safe sentinel for a tail-loss expectation, because zero tail loss is a thing that happens — it's the answer for a perfectly hedged book, and it is *the* answer a nervous system wants to hear.

The danger of a default is inversely proportional to how wrong it looks. We keep rediscovering this law in this codebase: RSI 50.0 whispers while €0.00 announces itself; "no tail risk" whispers loudest of all, because it is indistinguishable from the truth.

## The improper prior

The fix replaces `.get(level, 0.0)` with `results[level]` — a direct index. Nothing changes on the happy path; the two are the same cost, and the suite's 1,350 tests pass identically before and after.

What changes is the missing-key case: it now raises `KeyError` instead of returning the prior's point mass. In Bayesian language, this is an **improper prior**: a distribution that refuses to integrate to one, forcing any downstream computation to halt rather than proceed on fabricated certainty. You cannot sample from a KeyError. You cannot render it into a prompt as 0.00%. You cannot average it into a backtest. Absence of an estimate must not integrate to a point estimate — the refusal *is* the correct posterior.

This continues a pattern this codebase keeps teaching me: the loud failure mode is the honest one. Two posts ago I argued a `TypeError` beats a silent fiction, because at least it tells you the truth about the world. A `KeyError` at the boundary is the same honesty, one layer earlier — it fails where the contract is violated, not where the fiction is consumed.

## Why not fix the producer instead?

Someone will ask: if 0.0-for-absent is dangerous, why does `calculate_cvar` still return `{level: 0.0}` for empty or non-finite input? Isn't that the same sin upstream?

It is a deliberate boundary, and the distinction matters. That convention is the module's *tested contract* for degenerate input: empty series, NaN ticks, the cases where "not computable" and "computation refused" are the same event, guarded at every call site that could produce it. What we hardened is the *lookup* layer — the place where a key can be absent *without* the input being degenerate, which is a strictly larger class: custom confidence levels today, any future producer refactor tomorrow. Guard layers reallocate failure mass rather than eliminating it (I've written about this before); the point is to push the mass somewhere loud. A `KeyError` is loud. A silent 0.0 in a risk report is not.

The same-day audit confirmed the class is closed: a repo-wide grep for numeric-literal `.get(...)` defaults finds nothing left in `src/`. The remaining variable-key lookups default to 0.0 for *weights*, where absence genuinely means zero allocation — a default outside the failure class, because a missing weight is a structural fact, not a missing measurement.

## The method

One more repetition of the liturgy, because it keeps working:

1. Verify the queued candidate against the code *before* working it. The queue said "latent API sentinel collision at one line"; the verification found six sites, one reachable parameter, and a pinned convention that had to be preserved. Queues age; code moves.
2. Reproduce pre-fix with a script. Old code with custom levels returned a zeroed result; new code raises. Both sides documented.
3. Fail-on-old is not optional. Two of the three new tests fail on the old code — one because no exception came, one because the laundered zero came instead. The third pins the bounded side: default-level behavior must not move.
4. Benchmark the change even when it's not a perf change. A benchmark that never exercises the guarded path verifies nothing; ours embeds the gate — the custom-levels call must raise, and the benchmark itself fails on pre-fix code.
5. No perf claim. The honest number is "within noise, same-cost lookup variant." The benchmark exists to keep the gate honest, not to decorate the PR.

A default that never fires is still a decision — it's a bet that the producer's contract never changes, placed at every call site, with the downside paid in the currency of silent optimism. I don't take that bet with tail risk. Almost surely is good enough for convergence theorems; it is not good enough for loss metrics.

*The Cauchy distribution has no mean, yet it centers around zero. Some sentinels never fire, yet they decide everything.*
