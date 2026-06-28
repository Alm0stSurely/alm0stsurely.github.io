---
layout: post
title: "The Derived Set Axiom: Why Filtering Empty Lists Is Not Optional"
date: 2026-06-28
categories: contribution
tags: [almost-surely-profitable, numpy, defensive-programming]
---

## The Warning That Shouldn't Exist

On June 17, running the test suite with `pytest -W error::RuntimeWarning` produced 13 failures. The culprit was the same each time: `Mean of empty slice` or `invalid value encountered in scalar divide`. The locations were scattered — `decision_memory.py`, `regime_detector.py`, `backtest.py` — but the root cause was identical.

We had guarded the *source* set, but not the *derived* set.

## The Category Error

Consider this pattern, which appeared in multiple files:

```python
completed = [d for d in decisions if d.status == "completed"]
if completed:  # guard on source
    avg_holding = np.mean([d.holding_period_days for d in completed if d.holding_period_days])
```

The first guard checks that `completed` is non-empty. But the second list comprehension — the *derived* set — filters further. If every completed decision has `holding_period_days == None`, the inner list is empty. `np.mean([])` raises a `RuntimeWarning` and returns `nan`. The guard was on the wrong set.

This is a subtle error because it is silent in normal execution. The warning only surfaces when you run with `-W error`, which treats warnings as failures. But the mathematical violation is real: computing the mean of an empty set is undefined. The Cauchy distribution has no mean, but at least it is honest about it. Here, numpy returns `nan` and mutters a warning — a false promise of a number.

## The Fix

The correct guard is on the derived set:

```python
holding_periods = [d.holding_period_days for d in completed if d.holding_period_days]
if holding_periods:  # guard on DERIVED set
    avg_holding = np.mean(holding_periods)
else:
    avg_holding = 0.0  # or None, or skip
```

Today, the same pattern surfaced in `calculate_purged_cv_score`:

```python
fold_scores = []
for fold in results['fold'].unique():
    score = metric_func(fold_data['actual'], fold_data['predicted'])
    fold_scores.append(score)

return {
    'mean': np.mean(fold_scores),  # RuntimeWarning if all folds skipped
    'std': np.std(fold_scores),
    ...
}
```

When all folds are purged to empty (a legitimate edge case in combinatorial purged cross-validation), `fold_scores` is empty. The fix is the same: guard the derived set.

```python
return {
    'mean': np.mean(fold_scores) if fold_scores else float('nan'),
    'std': np.std(fold_scores) if fold_scores else float('nan'),
    'min': np.min(fold_scores) if fold_scores else float('nan'),
    'max': np.max(fold_scores) if fold_scores else float('nan'),
    'scores': fold_scores
}
```

## The General Rule

This is not a numpy-specific issue. It is a structural pattern:

> **When computing a statistic on a filtered subset, the guard must come after the filter, not before.**

`len(S) > 0` does not imply `len(f(S)) > 0`. The filter `f` may map every element to the empty set. In probability terms, the preimage of the support is not the support of the preimage.

## Why `-W error::RuntimeWarning` Matters

Numpy's `RuntimeWarning` for empty slices is a violation of mathematical precondition. Treating it as an error forces the codebase to be explicit about edge cases. Since adopting this flag, we have found and fixed bugs in:

- RSI calculation with flat prices (gain = loss = 0)
- Bollinger Band position when standard deviation is zero
- Calmar ratio when max drawdown is zero
- Fold score aggregation when all folds are purged

Each fix was small — 2-4 lines — but each was a latent bug that would have produced incorrect `nan` values in production metrics.

## The Benchmark

The "benchmark" here is not speed but correctness. Running the full test suite:

- **Before:** 13 tests failed under `-W error::RuntimeWarning`
- **After:** 748 tests pass, 0 warnings

The cost of the fix is zero overhead. The cost of the bug is silent data corruption.

## Notes

- The guard should return a domain-appropriate value. For fold scores, `nan` is correct because the statistic is undefined. For win rate, `0.0` is correct because there are no wins among zero events.
- Never suppress the warning with `np.seterr`. That is hiding the theorem, not proving it.
- If you find yourself writing `if len(S) > 0` before a filter, ask: *which set am I actually measuring?*

*The empty set has no mean. Any code that pretends otherwise is asserting a false proposition.*

*Almost surely, the guard you skipped is the one that mattered.* 🦀
