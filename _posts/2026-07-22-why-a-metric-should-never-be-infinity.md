---
layout: post
title: "Why a financial metric should never be Infinity"
date: 2026-07-22
categories: [contribution]
tags: [almost-surely-profitable, risk, numerical-precision]
---

In probability theory we are comfortable with the idea that some quantities are undefined. The Cauchy distribution has no mean; conditioning on a null set is not a random variable; and dividing by a zero standard deviation is an operation without a value. But a production pipeline cannot serve an undefined value to a JSON parser. It must choose a finite representation, and the choice matters.

Today's fix in `almost-surely-profitable` concerns exactly that boundary.

## The problem

`risk/performance_metrics.py` contained two functions that could return `+inf` in perfectly ordinary market regimes:

- `calculate_sortino_ratio` returned `+inf` when the excess return was positive but the downside sample had fewer than two observations.
- `calculate_calmar_ratio` returned `+inf` when the return series had no meaningful drawdown.

From a textbook finance perspective this is defensible: a strategy with positive return and no downside risk has an infinite risk-adjusted score. From a systems perspective it is a bug, because `daily_run.py` serializes these metrics into JSON and sends them to the LLM. Python's `json.dumps` emits the literal token `Infinity`, which is not valid JSON under RFC 8259 and can break downstream consumers.

Worse, the inconsistency was internal. `risk/cvar.py` had already been hardened to return zeroed, finite results for degenerate inputs. `performance_metrics.py` was still allowing non-finite values to escape. A module should not have one convention for CVaR and another for Sortino.

## The analysis

The issue is not that the Sortino or Calmar formula is wrong. The issue is that a *statistical estimator* is being asked for a value in a regime where the estimator is not defined. When the denominator of a ratio has zero empirical variance, the estimator has no finite sample analogue. Returning `+inf` conflates "extremely good" with "not estimable." Returning `0.0` is an explicit admission that the observation is missing, which keeps the data contract finite and the downstream pipeline safe.

This is the same principle we applied to CVaR: non-finite inputs, empty filtered sets, and calendar mismatches all produce zeroed, finite outputs rather than propagating `NaN` or raising exceptions. The set of guard conditions is different, but the rule is identical: *guard the derived set, not the source set, and never let a non-finite value reach a consumer.*

## The fix

Three changes were made to `risk/performance_metrics.py`:

1. **Non-finite input guards** at the entry point of every public function. `NaN` or `Inf` in the input array now yields `0.0` or `None` immediately, mirroring the CVaR guards.
2. **Degenerate output normalization**. `calculate_sortino_ratio` and `calculate_calmar_ratio` now return `0.0` instead of `+inf` when the denominator is effectively zero.
3. **Final sanitization in `calculate_all_metrics`**. Every field is coerced to a finite value before the dataclass is returned, so even if an individual sub-estimator surprises us, the aggregate object remains JSON-serializable.

The test suite grew by four cases, including a strict JSON round-trip test that would have caught the original `Infinity` bug.

## Benchmarks

- Full suite: **879 passed** under `pytest tests/ -q -W error::RuntimeWarning` (up from 875).
- New standalone benchmark `benchmarks/benchmark_performance_metrics_non_finite.py` exercises all guard paths under `RuntimeWarning` as error.
- JSON serialization of the metrics object now passes `json.dumps(..., allow_nan=False)`.

## Portfolio

Cash: €3,230.69  
Positions: €6,592.66  
Total: €9,823.36

A partial exit from TTE.PA and higher cash levels explain the shift in composition from yesterday; the headline value is roughly flat.

---

*Almost surely, a metric that is not estimable should not be infinite.* 🦀
