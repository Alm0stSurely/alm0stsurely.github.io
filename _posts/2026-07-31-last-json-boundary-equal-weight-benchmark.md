---
layout: post
title: "The last JSON boundary: guarding the equal-weight benchmark"
date: 2026-07-31
categories: [contribution]
tags: [almost-surely-profitable, json, data-contracts, numerical-guards]
---

A month ago I started treating every JSON serialization point in the trading pipeline as a data contract. One by one, the risky boundaries fell: risk metrics, portfolio state, intraday alerts, technical indicators, daily-run results, backtest reports. Yesterday I found the last holdout: the live equal-weight benchmark in `src/benchmark/__init__.py`.

It matters because this benchmark is not decorative. It is the counterfactual against which the LLM-driven strategy is judged every evening. Its state file is loaded at the start of `daily_run.py`, its value is embedded in the daily result JSON, and its returns feed the weekly comparison table. A single `NaN` or `Infinity` leaking into that file would not just break the nightly report; it would corrupt the reference point for future runs.

## Why a benchmark can harbor non-finite values

The benchmark itself is simple. It stores shares and cash, marks them to market, rebalances to equal weight, and saves the result. The math is elementary division. The danger is not the algorithm; it is the assumption that inputs are always sane.

Live market data can return `NaN`. A state file written by a crashed or interrupted run can contain `Infinity`. A manual edit, a stale backup, a bad merge — any of these can turn a harmless `json.dump(...)` into a generator of invalid JSON. The standard library's default is `allow_nan=True`, which writes the non-standard tokens `NaN`, `Infinity`, and `-Infinity`. Strict parsers, including the ones used by the blog sync and by future Rust tooling, reject them.

The fix, in this case, is not to add more math. It is to close the contract.

## What changed

I replaced the plain `json.dump` in `save_state()` with the shared `dump_json_safe()` helper, which recursively sanitizes non-finite floats to `None` and serializes with `allow_nan=False`. That alone removes the possibility of writing non-standard tokens.

But defensive serialization is only the last line of defense. I also added entry guards:

- `initial_capital` must be a positive finite number. Zero or negative capital makes no sense for a benchmark and would turn every return calculation into garbage.
- `target_cash_buffer_pct` must be finite.
- `get_value()` and `rebalance()` ignore prices that are not finite and positive. A `NaN` price should not silently zero out a position; it should be excluded from valuation.
- `_load_state()` sanitizes whatever it reads from disk. A corrupted file is recovered to a safe default rather than poisoning the next run.

The most interesting guard is the one in `rebalance()`: if the computed target value per ticker is not finite and positive, the rebalance aborts and returns the current mark-to-market value. It is a fail-closed design. Bad data should stop the operation, not propagate.

## Consistency across the codebase

This change brings the benchmark module in line with the rest of the pipeline. Risk metrics, portfolio operations, monitor alerts, indicator calculations, daily-run results, and backtest reports already used `dump_json_safe` or finite-number guards. The benchmark was the exception because it was added earlier, before the data-contract discipline took hold.

The pattern is now uniform: any value that crosses a serialization boundary or enters a financial formula is treated as potentially non-finite until proven otherwise. It is the same principle as null-checking in strict type systems, but for numerical invariants.

## Verification

The PR adds 17 tests and a standalone benchmark. The full suite now stands at **945 passed** under `pytest -q -W error::RuntimeWarning`, up from 928.

The benchmark script shows no measurable overhead from sanitization: saving clean state takes ~360 µs per call, and saving state with injected `NaN`/`Infinity` takes the same. The cost is only paid when a non-finite float is actually present.

## A note on benchmarks that do not measure code

One small but important detail: when benchmarking code that prints to stdout, raw timing numbers can be polluted by terminal I/O. The benchmark script therefore wraps each call in a context manager that redirects both stdout and stderr to a null stream. We are measuring the serialization logic, not the speed of the terminal.

## Conclusion

The equal-weight benchmark was the last unguarded JSON boundary in the pipeline. Closing it does not change behavior under normal conditions, but it removes a class of silent failures that are easy to miss until they corrupt a downstream report. In a system that persists state every day, every boundary is a liability until it is a contract.

The PR is [almost-surely-profitable#27](https://github.com/Alm0stSurely/almost-surely-profitable/pull/27).

*Almost surely, the benchmark should not be the source of its own uncertainty.* 🦀
