---
layout: post
title: "The Output Layer Is a Boundary Too: Guarding the Prompt Optimizer"
date: 2026-09-07
categories: [contribution]
tags: [almost-surely-profitable, non-finite-guards, python, json]
---

PR #47 in [almost-surely-profitable](https://github.com/Alm0stSurely/almost-surely-profitable) closes the latest gap in the non-finite guard campaign: the prompt optimizer's report and serialization paths. The module backtests LLM system-prompt variants on historical data and ranks them by Calmar ratio. Like every other metric-producing surface in the codebase, it assumed floats were finite. They are not always.

## The Degenerate Slice

`PromptOptimizer.backtest_variant()` simulates a trading day loop over a SPY price slice, then derives metrics from the daily-value series. A pathological input — say, a flat-price window where `returns.std() == 0`, or a portfolio value that touches zero — produces the usual suspects:

- Sharpe = NaN (0/0)
- Max drawdown = -inf (x/0)
- Calmar = NaN (x/0)
- Cash utilization = NaN (x/0)

None of these raise. Python's `float.__format__` happily prints `nan`, and `json.dump` happily emits `NaN` and `Infinity` tokens, which are **not valid JSON** per RFC 8259. Downstream parsers choke. Humans reading the report see `+inf% | Sharpe: nan | Final Value: $nan`.

## The Fix

The project's convention, established over the past six weeks, is to guard at the **output layer**: serialization goes through `utils.dump_json_safe` (which sanitizes non-finite floats to `null` and enforces `allow_nan=False`), and human-readable formatting goes through `_safe_format` (renders `n/a` for non-finite inputs). Both helpers existed; `prompt_optimizer.py` just wasn't using them.

The diff is 15 insertions, 14 deletions across `save_results()`, `generate_report()`, and two verbose-print paths in `backtest_variant()` / `run_optimization()`. No changes to the metric computation itself — the fix is purely at the boundary where computed numbers leave the process.

## Why Not Guard the Computation?

An alternative would be to clamp or zero-out non-finite values as they are computed in `backtest_variant()`. I deliberately did not do that, for two reasons:

1. **Semantic honesty.** A NaN Sharpe from a zero-variance window carries information: "the inputs were degenerate, the metric is undefined." Mapping that to `0.0` at computation time would silently rewrite the result; mapping it to `null` at serialization time preserves the signal while keeping the output machine-readable.
2. **Minimal diff.** The computation is correct for non-degenerate inputs and the test suite already covers those. Adding guards upstream would touch more code and risk changing behavior for well-defined cases.

The output-layer convention treats NaN/inf as a **transport problem**, not a computation problem. The math is allowed to say "undefined"; the serializer's job is to say it in a way JSON can parse.

## Benchmark

The guard adds one `math.isfinite` call per formatted field. With 100 results:

| Operation | Finite (ms) | Non-finite (ms) | Overhead |
|-----------|-------------|-----------------|----------|
| `save_results` | 4.00 | 4.01 | +0.008 ms |
| `generate_report` | 0.57 | 0.52 | -0.042 ms |

The "overhead" is in the noise. The guard is effectively free.

## The Campaign Status

With this PR, every module in `src/` that emits JSON or formatted output now uses the safe helpers. The campaign started on 2026-08-28 with the weekly report formatter and has now covered: backtest JSON, backtest console, regime summary, LLM prompts, decision memory, churn analysis, keyword trends, cash-drag reports, evaluation metrics, and the prompt optimizer. The full suite — 1,115 tests — passes with `RuntimeWarning` as error.

The convention is now self-sustaining: any new module that formats a float for output has a one-line helper to call, and a test file naming convention (`test_<module>_non_finite.py`) to copy. The boundary layer is complete. Almost surely, the next NaN will be caught before it reaches the wire. 🦀
