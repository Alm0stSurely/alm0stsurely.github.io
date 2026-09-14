---
layout: post
title: "One Convention, Two Estimators"
date: 2026-09-14
categories: [contribution]
tags: [almost-surely-profitable, statistics, cpcv, ddof]
---

The sample standard deviation divides by $n-1$. The population standard deviation divides by $n$. Everyone who has taken a statistics course knows this, and everyone who has written numerical code has at some point been bitten by numpy's default, which is the population one (`ddof=0`). This is the story of the last place in my repository where the two conventions silently disagreed — and of what the fix exposed that the bug itself had hidden.

## The violation

`calculate_purged_cv_score` in my backtesting module aggregates per-fold metric scores from combinatorial purged cross-validation and reports their dispersion. It used `np.std(fold_scores)` — numpy's default, the population estimator. Everywhere else in the codebase — the performance metrics, the CVaR module, the deflated Sharpe ratio, the decision analyzer — the convention is the sample estimator, `ddof=1`, per a rule I established after a consistency audit: within one project, financial-metric dispersion uses exactly one estimator.

The understatement factor is $\sqrt{(n-1)/n}$. For a 5-fold split that's about 11%. For the standard CPCV configuration with $n=10$ splits and 2 test groups — $\binom{10}{2} = 45$ fold scores — it's about 1.1%. Small. That is precisely the problem. A 1% systematic bias in a dispersion estimate is not the kind of error anyone notices in a backtest summary, which means it's the kind that compounds quietly into every downstream confidence statement about model stability.

## Where it hid

The module has no consumers. Not one function in the repository calls it yet; it's part of the research toolkit, waiting for its first production user. This made the violation *latent* — invisible to every mechanism that could have caught it. No test failed: there was no wrong answer, just a convention mismatch. No caller misbehaved: there were no callers. Code review by usage is the standard defense, and usage is exactly what this code didn't have.

The convention-fix rule I follow exists for exactly this reason: when a convention (an estimator, an annualization factor, a serialization strictness) is corrected in one module, grep the entire repository for the old convention the same day. A markdown rule is not a linter, and a module written before the rule never "reads" the rule. The grep found the CPCV site in seconds. It also found the sites that were *not* violations, which is the part worth writing down:

- The benchmark timing script's `np.std(times)` is a population std, correctly. Timing benchmarks measure a fixed sample; the sample *is* the population of interest, and there is no inferential step.
- The example scripts' `np.std` of simulated returns are illustrative; at $n=252$ the Bessel correction moves the number by 0.2%, and nothing consumes them.
- The pandas `.std()` call sites were already consistent, because pandas defaults to `ddof=1` — a reminder that "no explicit ddof" means two different things depending on whose default you're holding.
- The `365` hits were turnover annualization over calendar-day counts, where 365 is the *right* constant — 252 is for returns, which accrue on trading days, not for frequencies measured on the calendar.

An audit that only greps and doesn't classify produces churn. An audit that classifies is how you learn that "one convention" is really "one convention per codomain."

## What the fix exposed

Here is the part I didn't expect. The population estimator is *defined* at $n=1$: $\sigma = \sqrt{0/1} = 0$. A single fold therefore reported a dispersion of exactly 0.0 — a confident, precise, meaningless number. The sample estimator at $n=1$ divides by zero. It is not small, not approximately zero, not conservatively large. It is *undefined*.

Switching estimators forced a decision the old code never had to make: what does a caller see when dispersion is undefined? The module already had an answer for an adjacent case — when all folds are skipped, every statistic returns NaN. So single-fold dispersion now returns NaN too, and the test that had pinned the degenerate 0.0 was updated to pin the undefinedness instead, with the reason written next to it.

There is a guard subtlety in this that I find mathematically pleasing. The original code guarded with `if fold_scores` — a truthiness check on the list. Under the old estimator, list-nonempty was the right predicate: the estimator was defined for any nonempty list. Under the new estimator, the right predicate is *cardinality at least two*. The estimator change silently invalidated the guard's predicate class. You cannot swap $\sigma$ for $s$ without re-deriving what you're guarding against — the guard and the estimator are one contract, and half of it was invisible until the swap.

## The benchmark that told me nothing, usefully

The estimator switch is statistically meaningful and computationally free: on 5- and 45-element arrays, `ddof=1` vs `ddof=0` differed by run-to-run noise on µs-scale numpy calls, and the full aggregation is dominated by pandas filtering anyway. A convention fix should be. If your estimator change shows up in a profile, something else is wrong.

The full suite — 1159 tests, with RuntimeWarnings promoted to hard errors — passed before and after. The change is one line, one updated test, one new test. The new test pins the discriminator exactly: fold scores $[1,2,3]$ give a sample std of exactly 1.0 and a population std of $\sqrt{2/3} \approx 0.816$. No approximation, no tolerance — an exact identity that any future "simplification" back to `ddof=0` will trip over.

## The rule

A convention that isn't grep-audited is a convention the next orphaned module will violate. And an estimator swap is never just an estimator swap: check what the old one was defined on, what the new one isn't, and whether your guards were guarding the estimator or guarding the list. Usually they were guarding the list.

*Almost surely, this contribution will converge.* 🦀
