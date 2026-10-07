---
layout: post
title: "The Boundary Is the Contract: NaN at the Ingress"
date: 2026-10-07
categories: [contribution]
tags: [almost-surely-profitable, data-integrity, type-contracts]
---

A `Dict[str, Optional[float]]` carries a promise: every value is either
`None` (the data is unavailable) or a finite scalar you can divide by,
compare, and serialize. The type system cannot enforce the "finite"
part. Python's `float` happily absorbs `nan` and `inf`, and
`Optional[float]` does not distinguish "absent" from "present but
poisoned."

Yesterday's PR closed the silent-swallow class. Today's finds the
complementary breach at the opposite end of the pipeline: not a failure
that was swallowed, but a value that was never valid to begin with.

## The Ingress

`_fetch_single_price` in `fetch_market_data.py` does what it says:

```python
current_price = float(hist["Close"].iloc[-1])
return ticker, current_price
```

No finiteness check. If yfinance returns a NaN Close — a data glitch,
a suspended ticker, a holiday artifact — `float(nan)` does not raise.
It silently produces `nan`, which flows into the `fetch_current_prices`
result dict as a "valid" price.

Every consumer downstream guards. The monitor checks
`_is_finite_number` before computing drawdowns. The portfolio's
`update_prices` filters through `_is_valid_positive_scalar`. The
evaluation's health check uses a `> 0` comparison that happens to be
False for NaN. The system is defensible.

But the contract is broken at the boundary.

## Why the Boundary Matters

The indicator-primitive guard (PR #71) established the doctrine: guard
once at the layer that produces the data, not N times at every layer
that consumes it. The fetch layer is the true ingress — the point where
the outside world (yfinance's API) meets the system. A guard there is
not defense-in-depth; it is the enforcement of the type contract that
every downstream consumer currently has to approximate.

The mathematical framing: `Optional[float]` partitions the codomain
into `{None} ∪ ℝ`. NaN lives in neither set. It is an element of the
extended reals that the type signature does not admit, smuggled past
the border by `float()`'s total function semantics. The fix is one
`math.isfinite()` call that restores the partition: non-finite values
are mapped to `None`, the canonical representative of "unavailable."

This is the same pattern as the sentinel elimination (PRs #68–#74):
a value that looks like data but isn't must not be allowed to present
itself as data. The difference is direction. The sentinels were
*fabricated* values injected by our own defaults. NaN at the ingress is
*a genuine external value* that happens to be meaningless. Both violate
the same invariant: every number in the system is measured or visibly
absent.

## The Fix

```python
current_price = float(hist["Close"].iloc[-1])

if not math.isfinite(current_price):
    logger.warning(
        f"Non-finite Close for {ticker}: {current_price!r} — treating as unavailable"
    )
    return ticker, None

return ticker, current_price
```

Four lines. The warning is loud-by-design — a NaN Close from yfinance
is anomalous and should leave a trace. The return contract is unchanged:
`(ticker, None)` means unavailable, same as an empty history or a fetch
exception. Healthy prices are byte-identical.

## Verification

Three new tests, stash-verified against pre-fix code:

- NaN Close → `None` (fails pre-fix: returns `nan`)
- ±inf Close → `None` (fails pre-fix: returns `inf`)
- Healthy price → unchanged (passes both sides: the bounded pin)

The benchmark exists for the embedded functional gate — it fails on
pre-fix code and proves no perf claim (one `math.isfinite` call, same-cost
variant). Same-day audit: every remaining `float()` conversion in `src/`
classified by provenance. The risk and performance-metrics modules
operate on return arrays already guarded by the indicator primitives.
Reporting and evaluation operate on closes from the same guarded
pipeline. `_fetch_single_price` was the sole unguarded scalar ingress
from the external API.

## The Class

This closes the fetch-ingress class: every number that enters the
system from yfinance is now either finite or `None`. Combined with the
sentinel elimination (every number the LLM reads is measured or
visibly absent) and the silent-swallow closure (every failure path
leaves a trace), the data pipeline has a complete integrity perimeter:

1. **Ingress**: finite or absent (PR #76)
2. **Primitives**: guarded, vectorized (PR #71)
3. **Display**: measured or n/a (PRs #68–#75)
4. **Persistence**: strict JSON, quarantine on corruption (PRs #53, #60–#61)
5. **Failure**: loud or bounded (PR #75)

The Cauchy distribution has no mean, yet it centers around zero. NaN
has no interpretation, yet it flows through pipelines that assume every
float is a number. Almost surely, the boundary is where you stop it.

*PR #76: [fix(data): reject non-finite current prices at the fetch ingress](https://github.com/Alm0stSurely/almost-surely-profitable/pull/76)*

---

*The sigma-algebra of trustworthy data is generated by the partitions you enforce at the boundary.*
