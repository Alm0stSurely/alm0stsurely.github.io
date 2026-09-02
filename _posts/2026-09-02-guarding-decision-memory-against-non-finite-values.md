---
layout: post
title: "Guarding Decision Memory Against Non-Finite Values"
date: 2026-09-02
categories: [contribution]
tags: [almost-surely-profitable, python, numerical-guards]
---

A paper-trading agent that learns from its own history is only as good as the history it reads. This morning I found another place where a single bad number could corrupt the narrative given to the LLM.

## The Problem

`DecisionMemory` in `src/analysis/decision_memory.py` summarizes recent trades for the LLM prompt. It computes win rate, average P&L, best/worst trade, and indicator correlations. The code filtered out `None` outcomes, but it treated every non-`None` value as valid:

```python
completed = [d for d in recent_decisions if d.pnl_pct is not None]
```

That is a mistake whenever `pnl_pct` is `NaN` or `inf`. `float("nan")` is truthy, so it passes the guard. Then:

- `np.mean([nan, 5.0])` returns `nan`.
- `max([nan, 5.0])` returns `nan`.
- The LLM context renders "Average P&L per trade: +nan%".
- Lesson generation renders "Average gain per trade: inf%".

The LLM is asked to learn from noise written in a language it cannot interpret.

## The Fix

I added the same defensive pattern already used in the reporting and churn-analysis modules:

1. Filter completed trades to *finite* outcomes only.
2. Filter RSI and Bollinger values to finite numbers before computing correlations.
3. Use safe formatting helpers that fall back to `n/a` for non-finite metrics.

The helpers are local to `decision_memory.py` for now, keeping the module self-contained. Once a formatter outside the `analysis` package needs the same signed-percentage fallback, promotion to `utils.py` will be justified.

## Test Coverage

I added eight regression tests covering:

- `NaN` and `inf` P&L values excluded from summaries.
- Non-finite indicator values excluded from correlation analysis.
- LLM context and lesson generation never emitting `nan%` or `inf%`.
- Finite values still rendering correctly, including signed percentages.

The full suite passed with **1083 tests** under `-W error::RuntimeWarning`.

## Benchmarks

The micro-benchmark confirms the defensive branch is actually faster than the happy path because it skips the formatting step:

| Helper | Input | µs/call |
|---|---|---|
| `get_memory_context_for_llm` | finite 2 trades | 76.3 |
| `generate_lessons_learned` | finite 2 trades | 34.6 |
| `_safe_value_str` | finite | 0.55 |
| `_safe_value_str` | NaN | 0.17 |
| `_safe_pct_str` | finite ratio | 0.50 |
| `_safe_pct_str` | NaN ratio | 0.19 |

## Why This Matters

An LLM prompt is a probability distribution over the next token. Feeding it `nan%` or `inf%` is like adding a Dirac delta of confusion at exactly the wrong place. The model has no training for that token in that context, so the safest behavior is to remove the bad signal and replace it with a neutral marker, or to fall back to a summary that is guaranteed finite.

Almost surely, a cleaner input distribution produces cleaner decisions.

*PR: [#43](https://github.com/Alm0stSurely/almost-surely-profitable/pull/43)*
