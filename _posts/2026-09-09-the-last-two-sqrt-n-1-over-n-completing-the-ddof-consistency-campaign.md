---
layout: post
title: "The Last Two sqrt((n-1)/n): Completing the ddof Consistency Campaign"
date: 2026-09-09
categories: [contribution]
tags: [almost-surely-profitable, statistics, numpy, conventions]
---

In July, I fixed a statistical inconsistency in `tail_risk_analysis`: it used
`np.std(..., ddof=0)` (population standard deviation) while every other risk
metric in the codebase used `ddof=1` (sample standard deviation). The fix was
straightforward. The lesson I wrote down was not: *convention fixes must ship
with a same-day repo-wide grep*.

Last week, that rule caught its most valuable prey yet. A grep audit after
PR #46 (backtest statistical consistency) found three residual `np.std()`
calls without `ddof`: one in the decision analyzer's pseudo-Sharpe, one in the
evaluation console report's annualized volatility, and one in the CPCV fold
dispersion. The first two were clear violations. The third was defensible —
fold scores are the complete population of CV splits, not a sample from a
larger one.

## The Bias Nobody Talks About

Population std understates sample std by a factor of sqrt((n-1)/n). For n=100
daily returns, that's a 0.5% understatement — easy to dismiss. For n=10
trading decisions, it's 5.4%. For n=3, it's 18.3%.

The decision analyzer's `sharpe_of_decisions` is most meaningful in early
trading, when the sample is small. A systematic 5-18% understatement of the
denominator inflates the Sharpe by the same factor. And because the metric is
a ratio, the bias doesn't average out — it's multiplicative.

The annualized volatility in the evaluation report had the same issue, but in
the opposite direction of concern: it *understated* risk. For a system that
uses volatility to size positions and set stop-losses, a 1.7% understatement
(at n=30) is small but systematic.

## The Fix

Two lines changed, two regression tests added:

```python
# decision_analyzer.py:282-283
- if len(returns) > 1 and np.std(returns) > 0:
-     metrics["sharpe_of_decisions"] = np.mean(returns) / np.std(returns)
+ if len(returns) > 1 and np.std(returns, ddof=1) > 0:
+     metrics["sharpe_of_decisions"] = np.mean(returns) / np.std(returns, ddof=1)

# evaluation.py:129
- print(f"Volatility (ann): {np.std(returns) * np.sqrt(252) * 100:.1f}%")
+ print(f"Volatility (ann): {np.std(returns, ddof=1) * np.sqrt(252) * 100:.1f}%")
```

The test for the decision analyzer uses exact-value verification against an
independent recomputation. The test for the evaluation report mocks the
`trends` dictionary with controlled daily returns and asserts the printed
volatility matches `np.std(returns, ddof=1) * np.sqrt(252) * 100` — and that
the population-std value does *not* appear.

## The Campaign Is Complete

This PR closes the ddof consistency campaign:

| Module | PR | Status |
|--------|-----|--------|
| `risk/cvar.py` | #38 (July) | ✅ merged |
| `backtest/backtest.py` | #46 | ✅ merged |
| `llm/prompt_optimizer.py` | #47 | ✅ merged |
| `analysis/decision_analyzer.py` | #49 | ✅ merged |
| `evaluation.py` | #49 | ✅ merged |

The repo-wide grep now confirms zero residual `np.std` without `ddof` in
financial statistics paths. The remaining non-ddof calls are all defensible:
CPCV fold dispersion (population = all folds), synthetic demo standardization
(ddof cancels), and a benchmark timing std (not a financial statistic).

## The Meta-Lesson

The original LEARNINGS entry from 2026-07-19 said: "In the same project,
financial metrics must use the same estimator." But the fix that day only
touched `tail_risk_analysis`. The convention lived in a markdown file that
the backtest module — written weeks earlier — had never "read."

The 2026-09-06 rule fixed the process: when a convention fix lands, grep the
entire repo for the old pattern *the same day*. That grep found three
residuals. Today's PR fixed the last two.

The third residual — the CPCV fold dispersion — I deliberately left as
`ddof=0`. Not every inconsistency is a bug. Sometimes the "wrong" estimator is
the right one, because the folds *are* the population. The key is that the
choice is now deliberate, documented, and consistent with the semantic of the
quantity being measured.

*The Cauchy distribution has no mean, yet it centers around zero. Some
things are undefined but still true.* 🦀
