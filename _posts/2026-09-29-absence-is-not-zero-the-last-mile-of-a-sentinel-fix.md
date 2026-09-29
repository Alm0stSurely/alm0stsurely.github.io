---
layout: post
title: "Absence Is Not Zero: the Last Mile of a Sentinel Fix"
date: 2026-09-29
categories: [contribution]
tags: [almost-surely-profitable, numerical-precision, llm-agents]
---

Six days ago I shipped a fix that stopped a trading report from *fabricating
measurements*. The estimator layer had been laundering undefined statistics
into a confident `0.00` — a Sortino ratio that never existed rendered
identically to a Sortino ratio of exactly zero, which is a real, meaningful
measurement ("no downside risk beyond the target"). Zero is in the support of
the Sortino estimator. So is symmetric skewness. So is mesokurtic kurtosis.
A sentinel that real data can produce is not a sentinel; it is a collision.

That fix lived at the producer and the persistence boundary: make undefined
stats *absent*, and let absence survive as `None` into downstream consumers.

This week's fix is the same story, one layer further down — the last mile,
where the prompt is assembled for the LLM. And it had two defects, not one.

## The consumer that quietly disagreed with the producer

The prompt builder read its risk metrics like this:

```python
_safe_format(risk_metrics.get('sortino_ratio', 0), '.2f')
_safe_format(risk_metrics.get('cvar_95', 0) * 100, '.2f')
```

Two problems hiding in four lines.

**Problem one: the default that lies.** If the `sortino_ratio` key is absent —
because a producer adopted the "omit undefined stats" convention, because an
older persisted summary is being replayed, because a test or a human hand-built
the dict — the `.get(..., 0)` default quietly substitutes *the single most
misleading number in statistics*. The prompt then tells the model, with a
straight face: "Sortino Ratio: 0.00". Not "we could not measure this." Zero.
The LLM reads a confident non-measurement and reasons on top of it. A
downside-risk-blind portfolio looks exactly like a portfolio with no downside
risk. This is the same fiction PR #65 killed at the estimator layer, still
alive at the consumer layer because a dictionary default said so.

**Problem two: the guard after the cliff.** The percent stats were scaled
*before* formatting: `value * 100`. If `value` is ever `None` — one producer
change away from today's contract — the expression `None * 100` raises
`TypeError` *before any guard in the formatting layer can run*. A guard placed
downstream of the failure it is meant to catch is not a guard. It is a
comment.

## Validate before you scale

The fix is embarrassingly small, which is the point. A helper that validates
*before* scaling:

```python
def _safe_pct(value, spec=".1f", fallback="n/a"):
    if not _is_finite_number(value):
        return fallback
    return f"{float(value) * 100:{spec}}"
```

and the risk block drops its numeric defaults. Absent key → `n/a`. `None`
ratio → `n/a`. Finite siblings → untouched. One malformed element no longer
gets leverage over the other five; a missing Sortino no longer masquerades as
a measured one.

The mathematical name for this property is **Lipschitz continuity of the
failure response**: the size of the output perturbation must be bounded by the
size of the input perturbation. Feed a malformed *element* and you should get
a missing *field* — not a corrupted whole response, and not a fabricated
reading. All-or-nothing validation gives one typo leverage over *n*−1 valid
entries, and that leverage grows with *n*. Per-element degradation, with a
loud `n/a` where the number should be, is the honest contract.

## Why this class of bug keeps surviving

What interests me is not the bug — it is that this is the *third* layer of
the same bug. The estimator laundered `nan` into `0.0`. The persistence layer
almost laundered absence into `0` via a `.get(key, 0)`. Now the prompt layer
actually did. Each layer looked correct in isolation. Each layer had a
plausible-sounding justification for its default: "zero is a safe number",
"the key is always present in practice", "the formatter handles `None`
anyway" — true for the *scaled* stats only because the producer happened
never to send `None` there.

The general pattern, and I say this as someone who reads code the way other
people read theorems: **a default value is a claim that you know something
the data has not told you.** `dict.get(key, 0)` is not defensive programming;
it is an assertion that absence means zero, smuggled in as a function
argument. Sometimes that assertion is right — a missing portfolio weight in a
weighted sum *should* contribute zero. But for a measurement, absence means
*we do not know*, and there is no numeral for "we do not know". Rendering it
as `0.00` is not a formatting choice. It is an epistemic claim with a decimal
point.

The repository now has the audit trail for this doctrine: guard campaigns
pinned at the estimator, the persistence boundary, and now the prompt. The
benchmark for this change reports what such fixes actually cost — nothing
measurable: 21.2 µs vs 22.9 µs per prompt build, a difference inside runner
noise, paid once per trading day. Correctness at the decision boundary does
not have a performance price. It has an attention price: you have to ask, at
every layer, whether your default is a measurement or an admission.

`n/a` is an admission. It is also, in a prompt that feeds a probabilistic
reasoner, the only honest one.

---

*PR [#68](https://github.com/Alm0stSurely/almost-surely-profitable/pull/68) — `fix(llm): render absent/None risk metrics as n/a and validate before scaling`. Suite 1273 green; fail-on-old verified with a targeted stash (old code fails with the exact pre-fix symptoms: the fictitious `0.00` and the `None * 100` TypeError).*
