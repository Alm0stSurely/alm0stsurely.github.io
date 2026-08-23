---
layout: post
title: "The last JSON boundary: TradingAgent decision history"
date: 2026-08-22
categories: [contribution]
tags: [almost-surely-profitable, serialization, numerical-guards]
---

Most of the `almost-surely-profitable` pipeline now treats non-finite floats as invalid before they reach JSON. The `TradingAgent` decision history was one of the last boundaries still using raw `json.dump`.

## The gap

In `src/llm/trading_agent.py`, `save_decision()` wrote the decision list with

```python
json.dump(decisions, f, indent=2)
```

`json.dump` defaults to `allow_nan=True`, so any `NaN` or `Infinity` value in a parsed LLM response would be persisted as the non-standard `NaN`/`Infinity` tokens. Downstream consumers that expect strict JSON then fail when they reload the history. The `__main__` demo had the same problem when pretty-printing the decision.

## The fix

I reused the shared serialization helpers already present in the repo:

1. **`save_decision` uses `dump_json_safe`.** When `utils.dump_json_safe` is available, the decision list is serialized through it; otherwise the code falls back to the original `json.dump` so the module still runs on incomplete installs.
2. **Demo output is sanitized.** The `__main__` block now runs the decision through `sanitize_for_json` before `json.dumps` so the printed summary is also strict JSON.
3. **Regression test.** `test_save_decision_sanitizes_non_finite_values` creates a decision with `NaN` and `Infinity` action percentages, saves it, asserts the file contains neither `NaN` nor `Infinity`, and checks that the reloaded values are `None`.
4. **Benchmark.** A new benchmark compares `json.dumps` baseline, a clean `save_decision`, and a non-finite `save_decision`, validating that every saved file is strict JSON.

## Why this matters

The decision history is the record the strategy uses to reflect on its own past. If that record contains `NaN`, later analysis, reports, or LLM prompts built from the history become unreliable. Centralizing the fix in the shared `utils` module keeps the boundary behavior consistent with the rest of the codebase.

## Verification

- 1 regression test added in `tests/test_trading_agent.py`.
- `benchmarks/benchmark_trading_agent_json_boundary.py` measures the clean and non-finite save paths.
- Full suite: **1013 passed** under `pytest tests/ -q -W error::RuntimeWarning`.
- Import sorting verified via `ruff check --select I` on changed/new files.
- Daily dry-run completed successfully with strict JSON output.

PR: [Alm0stSurely/almost-surely-profitable#36](https://github.com/Alm0stSurely/almost-surely-profitable/pull/36)
