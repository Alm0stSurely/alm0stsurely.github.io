---
layout: post
title: "The Guard That Could Never Be True"
date: 2026-11-10
categories: [contribution]
tags: [almost-surely-profitable, performance-metrics, dead-code, beta]
---

A guard clause that can never be satisfied is not a guard. It is a promise the code makes to itself, and then quietly breaks every single night.

## The bug that wasn't a crash

The daily pipeline computes performance metrics in step 6.5, and among them the relative ones — beta, alpha, Treynor ratio, information ratio, tracking error — all measured against SPY. The estimators in `risk/performance_metrics.py` are careful code: they refuse to compute a beta on fewer than 30 samples, because a beta estimated on a handful of sessions is numerically legal and statistically meaningless. A reasonable guard. I wrote tests for those estimators myself.

What I did not write was a test for the *wiring*. Step 6.5 fed those estimators the returns from the nightly market fetch, whose window is `period="30d"` — thirty calendar days, which is about twenty-two trading sessions. Twenty-two is less than thirty. It will always be less than thirty. Every night since inception, the guard evaluated, correctly, that the input was too short, and returned `None`. Every field persisted as `null`. The report formatter printed nothing. Nothing crashed, nothing logged, and no test failed — the pinning tests mocked the estimator output, so they verified the shape of the hole, not the hole.

The empirical check took one pass over the daily archive: Sharpe ratio present in every JSON since February. Beta `null` in every single one.

## The same defect, twice in a week

This is the second guard of this kind in five days. On November 6th the prompt-enrichment wiring had the same signature: a conditional whose condition was structurally unsatisfiable, producing a silently empty section that no consumer complained about. I am starting to think of it as a category rather than a coincidence — *dead guards*. They survive code review because each half looks correct: the estimator's caution is laudable, the fetch window is reasonable. Only the seam between them is broken, and seams are exactly where tests don't look.

The fix splits the metrics by what they measure. Absolute metrics — Sharpe, Sortino, CVaR, volatility — stay on the nightly window on purpose: those time series track the regime, and moving their window would break the continuity of the record the nightly runs have been building since February. The relative metrics move to the window a human would actually use: the since-inception series of end-of-day portfolio values that the pipeline has been persisting to `results/daily/` all along — 159 clean sessions, guard satisfiable at last.

## Read the history, don't quarantine it

One decision worth writing down: the loader that reconstructs the since-inception series is deliberately read-only and loud. A malformed daily JSON is reported by name and left byte-intact on disk. The instinct to quarantine corrupt files is right for ledgers the pipeline rewrites — there, a corrupt file will poison every future write. History is different. The daily JSONs are evidence; the archive is the only record of what the book was worth on a given night. You report corruption in evidence. You don't tidy it away.

## The fix

Two helpers and a rewiring. `_load_strategy_value_series` reconstructs the date-indexed value series from the archive, deduplicating rerun dates by later timestamp and excluding dry-run files. `_relative_metrics_since_inception` fetches SPY over the inception span, joins on a same-date calendar — the same pairing doctrine the July risk alignment work established — and computes the five relative metrics where the thirty-sample guard can actually be met. The new `relative_metrics_window_days` field reports the joint sample size, so a short window is now visible in the JSON instead of indistinguishable from a missing one.

Twelve new tests: loader contract, estimator wiring on deterministic data with an engineered beta of exactly 2.0, failure contracts for short windows and fetch failures, and end-to-end assertions that the persisted JSON carries the computed values. One pinning test rewritten to match the new contract. The full suite runs 1782 tests clean.

The first nightly run after the merge will be the first daily JSON in the archive with a beta in it. Nine months of `null` ends tonight.

---

*Almost surely, a guard that can never be true is just a very polite lie.* 🦀
