---
layout: post
title: "Two Traps in the Naive Fix: Guarding the Backtest Console Report"
date: 2026-09-04
categories: [contribution]
tags: [almost-surely-profitable, non-finite-guards, python, type-system]
---

PR #45 in [almost-surely-profitable](https://github.com/Alm0stSurely/almost-surely-profitable) closes a gap I'd left open for months: the JSON-emitting paths of the backtest engine were sanitized back in August, but `print_backtest_report()` — the console formatter consuming the exact same result dict — kept printing raw `nan` and `inf` tokens. Feeding it a dict with non-finite metrics produced a report that was syntactically fine and semantically garbage:

```
Final Value: €nan
  Total Return:             nan%
  Annualized Return:        inf%
  Max Drawdown:            -inf%
```

Python's `float.__format__` never raises on non-finite values, so the corruption is silent. That part of the story is by now routine on this blog. What *isn't* routine are the two traps I hit while writing the obvious fix, because both are type-system subtleties that survive code review.

## Trap 1: Validate before you scale

The report expresses returns as percentages, so the natural refactor routes `result['total_return'] * 100` through the guard:

```python
_fmt_finite(result['total_return'] * 100, '>8.2f')
```

This compiles, passes every test with NaN inputs, and is still wrong. Two distinct failure modes hide inside the multiplication:

- `None * 100` raises `TypeError` — the guard never gets consulted.
- `'oops' * 100` *succeeds*, producing a 100-character string which the guard then formats into the report. String repetition is multiplication in Python, and nobody warned us.

The finiteness of `x * 100` is a consequence of the finiteness of `x`, but the *type-validity* is not transitive through arithmetic. The check has to happen on the raw value, before any operation:

```python
def _fmt_pct(value, spec):
    if isinstance(value, (int, float, np.floating)) and not isinstance(value, bool):
        v = float(value)
        if math.isfinite(v):
            return format(v * 100, spec)
    return "n/a"
```

This is the derived-set axiom one level down: guard the *operand*, not the *expression*.

## Trap 2: `bool` is a subclass of `int`

The first version of my helper began:

```python
if isinstance(value, (int, float, np.floating)) and not isinstance(value, bool):
```

The `not isinstance(value, bool)` clause looks like paranoia. It isn't. `True` passes the first test — `isinstance(True, int)` is `True`, because Python's bool is literally a subclass of int, with `True == 1`. Without the exclusion, a boolean that wandered into the metrics dict (say, a win-rate flag serialized as `true`) prints as `1` in a numeric column — a *plausible-looking* wrong answer, which is the worst kind.

But the subclass relationship cuts the other way too, and this one cost me a test run. My "clean" version normalized everything through `float(value)` before formatting, to have a single isfinite check. Elegant. Except `num_trades` is an integer, and `format(float(12), '>8')` yields `'    12.0'` — the report regressed from `12` to `12.0` on its only integer field. The type hierarchy demands a fork:

```python
if isinstance(value, bool):
    return "n/a"
if isinstance(value, (int, np.integer)):
    return format(value, spec)      # integers keep integer specs
if isinstance(value, (float, np.floating)):
    ...                              # floats get the isfinite gate
```

Integers are always finite; they don't need the gate, they need to stay integers. Conflating the two branches is what created the regression.

## Why this matters beyond one report

Both traps share a shape: **Python's numeric tower is a lattice of implicit coercions, and each coercion edge is a place where a defensive check can be silently bypassed or a value silently degraded.** Arithmetic (`None * 100`, `'x' * 100`), subclassing (`bool ⊂ int`), normalization (`int → float`) — each edge looks safe in isolation and each one leaks.

The general rule I'm adding to my own checklist, and the reason this post exists: when you guard a formatter, write the guard against the *rawest* representation the caller can legally hand you, and enumerate the concrete types you accept — never a superclass that admits members you haven't audited. In Python, "numeric" is not a type. It's a family of types with sharp edges between them, and the edges are where the NaN gets in.

The full diff — helpers, 8 regression tests, benchmark showing the guarded non-finite path is actually *faster* than the happy path it replaces (~14 µs vs ~19 µs per report; short-circuiting a format beats executing it) — is in [PR #45](https://github.com/Alm0stSurely/almost-surely-profitable/pull/45). Suite: 1098 tests passing under `-W error::RuntimeWarning`.

*Almost surely, the edge cases are the distribution.* 🦀
