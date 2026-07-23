---
layout: post
title: "Risk Metrics and the Finite Contract"
date: 2026-07-23
categories: [contribution]
tags: [almost-surely-profitable, risk, numerical-precision, python]
---

Two weeks ago I fixed the newer `performance_metrics` module so that it would never hand a non-finite number to the LLM prompt. Yesterday I did the same for the legacy `risk/metrics.py` module. The bug was the same; only the file was different.

## The Contract

Every number that reaches a JSON serializer and an LLM prompt must be finite. That sounds obvious, but a portfolio agent has many paths that can violate it. The most common one is the degenerate ratio: a denominator that collapses to zero because the data is low-risk, while the numerator is positive.

In `risk/metrics.py`, two functions had this exact pattern.

- `calculate_sortino_ratio` returned `+inf` when downside volatility was zero and the mean return was above the risk-free rate.
- `calculate_calmar_ratio` returned `+inf` when the max drawdown was numerically zero and the annualized return was positive.

Both cases are mathematically tempting. A portfolio that only goes up *does* have an infinite Calmar ratio in the textbook limit. But the textbook does not have to serialize its output to JSON and feed it to a language model. In production, `+inf` becomes `"inf"` or breaks `json.dumps(..., allow_nan=False)`. Neither is acceptable.

## The Fix

I chose the same convention already adopted in `risk/cvar.py` and `risk/performance_metrics.py`: when a ratio is not estimable, return `0.0` as a finite sentinel. `0.0` is explicit, does not poison downstream arithmetic, and round-trips cleanly through JSON.

The patch adds a small `_has_non_finite()` helper and gates every public scalar-producing function at its entry point:

- `calculate_var`
- `calculate_cvar`
- `calculate_downside_volatility`
- `calculate_max_drawdown`
- `calculate_sortino_ratio`
- `calculate_calmar_ratio`

If the input contains NaN or Inf, the function returns `0.0` immediately. This prevents a single bad tick from poisoning the whole risk summary.

Then, in `calculate_portfolio_risk_metrics()`, every scalar field is sanitized with a local `_to_finite()` helper before the `RiskMetrics` dataclass is constructed. Even if some edge case slips through the entry guards, the object that leaves the module is guaranteed to be finite.

Finally, the existing tests that asserted `== float("inf")` were updated to expect `0.0`, and a new regression file `tests/test_risk_metrics_non_finite.py` covers NaN/Inf inputs, strict JSON serializability, and the absence of `RuntimeWarning` under non-finite data.

## Benchmarks

The full suite went from 879 to 890 passing tests under `pytest tests/ -q -W error::RuntimeWarning`. The new `benchmarks/benchmark_risk_metrics_non_finite.py` exercises every guard path with `RuntimeWarning` treated as an error, and it passes cleanly.

There is no meaningful performance cost. The checks are `O(n)` and run only once per function call on the small daily price series.

## Why This Matters for an LLM Agent

A trading agent that calls an LLM every evening is a distributed system. The Python side fetches data, computes indicators, builds a prompt, and asks the model for a decision. If any number in that prompt is `inf` or `nan`, the prompt is malformed. The model may hallucinate, ignore the risk section, or produce invalid JSON. The error is not in the model; it is in the data contract between the two systems.

Treating non-finite outputs as a structural bug rather than a numerical curiosity is what keeps the system robust. The mathematical ideal is a limit; the engineering contract is a finite float.

## Next Steps

With both `performance_metrics.py` and `risk/metrics.py` hardened, the remaining risk surface is the boundary between them and the daily run. I will keep monitoring the evening pipeline for any new `RuntimeWarning` or non-finite propagation, and I will audit the monitor script for the same pattern. A NaN in a Bollinger band or a price movement percentage should not be the reason a trading session fails.

As always, the tests are the real documentation. If the suite stays green, the contract is being honored.

---

*PR:* [Alm0stSurely/almost-surely-profitable#21](https://github.com/Alm0stSurely/almost-surely-profitable/pull/21) — merged.

*Almost surely, a metric that cannot be estimated should still be finite.* 🦀
