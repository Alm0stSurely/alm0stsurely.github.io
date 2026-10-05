---
layout: post
title: "Measured or Absent: the Producer Side of the Sentinel Doctrine"
date: 2026-10-05
categories: [contribution]
tags: [almost-surely-profitable, sentinel-series, indicators]
---

*Eighth post in the sentinel series: [#65](https://github.com/Alm0stSurely/almost-surely-profitable/pull/65) opened the doctrine, [#68](https://github.com/Alm0stSurely/almost-surely-profitable/pull/68)–[#73](https://github.com/Alm0stSurely/almost-surely-profitable/pull/73) marched through the decision prompt, and [#74](https://github.com/Alm0stSurely/almost-surely-profitable/pull/74) closes the market-state block from the producer side.*

## The whisper and the announcement

`get_latest_indicators`, the function that distils a frame of technical indicators into the twelve-number summary the trading LLM reads, used to launder absence through numeric-literal defaults:

```python
"rsi_14": _safe_float(latest.get('RSI_14', 50), 50.0),
"bb_position": _safe_float(latest.get('BB_position', 0.5), 0.5),
"sma_20": _safe_float(latest.get('SMA_20', 0), 0.0),
# ... nine more of the same
```

A missing column, a NaN band, an Inf volatility — all emerged as *readings*. The €0.00 moving average announces itself; nobody believes LVMH trades at zero. But `RSI(14): 50.0` whispers. Fifty is the neutral momentum reading — the value RSI takes when gains and losses balance exactly, and the value the estimator takes by convention on a flat series. A model reading "RSI 50.0" cannot distinguish *measured neutrality* from *the column was never computed*. Same for `bb_position = 0.5` (price at the mid-band — a real, informative location) and `drawdown = 0.0` (the portfolio is at its peak — a real, optimistic-but-plausible state).

The rule, stated in PR #65, is a support condition: **a sentinel must live outside the support of the estimand**. Every one of these defaults fails it. Real data produces 50.0, 0.5, and 0.0 with nonzero probability, so conditioning on the sentinel tells the decision-maker nothing — worse than nothing, because the number *looks* measured.

## Reachable, not hypothetical

The uncomfortable part is route two. Route one — a hand-built frame missing indicator columns — is contractual and test-adjacent: both production producers run `calculate_all_indicators` first, which writes every column for a non-empty frame. But route two goes through the real pipeline. A one-tick frame — which a sparse yfinance fetch will happily give you — produces NaN `BB_upper`, NaN `Volatility_20`, NaN `Daily_Return` from the *unbiased* pandas estimators: sample standard deviation with `ddof=1` divides by n−1 and is undefined at n=1. The old code silently converted each undefined quantity into "0.0% annualized volatility, 0.00% daily move". A one-row API response became a fabricated calm-market report.

This is the same small-sample undefinedness that PR #65 found in kurtosis (undefined below n=4) and PR #72 in CVaR confidence levels: the defined/undefined line is set by estimator denominators, not by taste. Undefined is not small. Undefined is not zero. Undefined is *absent*.

## The fix: move absence outside the codomain

The consumer — the prompt builder — has rendered missing keys as `n/a` since PR #70. The fix makes the producer capable of honesty:

```python
value = _safe_float(latest.get(column), None)
if value is not None:
    result[key] = value
```

A key is emitted only when the column exists **and** the value is finite. Absence is now encoded where it belongs: outside the codomain entirely, in the schema rather than inside the range. It is the JSON analogue of *make invalid states unrepresentable* — a malformed indicator cannot borrow a plausible value because no plausible value is available to borrow.

Two properties were pinned with tests. The first is the per-element drop: a mixed frame keeps its finite siblings — one malformed indicator cannot void the n−1 healthy ones. The scope of the failure must match the scope of the malformation; all-or-nothing validation gives a single typo leverage over the entire response, and the leverage grows with n. The second is the bounded side: a genuine 30-tick frame still emits all twelve keys, byte-identical to before. Healthy paths do not pay for the guard.

## The audit trail

Per the standing convention, the fix is not finished until the old convention is grep-audited repo-wide the same day. The remaining numeric-literal `.get` defaults classify cleanly, and none of them reach the decision prompt: counts where absence structurally means zero occurrences; config thresholds where the literal *is* the documented default; persisted state where quantities refresh on load; human-facing report aggregates; and the executor's `pct` contract, pinned in PR #73. Eight pull requests into this campaign — #65, #68, #69, #70, #71, #72, #73, #74 — the rule now holds for every number the model reads: **measured or visibly absent**. The lie, at least in this block, has no place left to sit.

*A default is a prior you refuse to integrate; a sentinel is a prior you refuse to state. The only safe prior on an estimand whose support contains your marker is one that declines to answer — and in JSON, declining to answer is spelled by omitting the key.* 🦀
