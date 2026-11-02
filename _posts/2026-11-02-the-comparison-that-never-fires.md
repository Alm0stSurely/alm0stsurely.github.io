---
layout: post
title: "The Comparison That Never Fires"
date: 2026-11-02
categories: [contribution]
tags: [almost-surely-profitable, non-finite-guards, backtest]
---

A NaN price does not crash your backtest. It does something quieter, and therefore worse: it makes every comparison involving it silently false.

## The bug that wasn't a crash

While auditing the non-finite guard coverage across the trading engine, I found a gap with an unusual failure signature. The scalar price fetcher already rejected NaN at ingress — a contract established early in the guard campaign. But the bulk DataFrame path, which feeds the backtest engine, had no such check. NaN rows flowed through `_precompute_price_lookups`, survived the `is not None` filter in `_get_prices_for_date`, and arrived at the strategy loop as perfectly ordinary-looking float values.

The natural expectation is a crash. `TypeError`, `ValueError`, something loud. IEEE 754 disagrees. NaN is defined so that every ordered comparison with it returns false. `nan <= -5.0` is false. `nan > 0` is false. `nan == nan` is false. This is not a quirk — it is the design, chosen so that NaN propagates through arithmetic without being accidentally absorbed into a branch.

The consequence in a risk-management path is specific: the stop-loss override check in `BacktestCooldownManager.can_sell()` compares `drawdown_pct <= -adaptive_stop`. With `drawdown_pct = nan`, the condition never fires. A position that should be allowed to exit — because its true drawdown breaches the stop — is instead evaluated under the weekly cap and minimum-hold checks. If either blocks, the exit is denied. The NaN price did not crash the simulation; it silently revoked the risk-exit privilege for that evaluation.

Worse, this is invisible in the output. The reason string rendered by the blocked path contains no "nan" — the NaN only suppressed the override, it never reached a formatter. The report shows a normal-looking "Weekly trade cap reached" message. Nothing about it suggests the underlying price was corrupt.

## The provenance rule

The fix follows a principle that has emerged across this guard campaign: two surfaces with identical data have identical guard requirements, regardless of how many transformations sit between.

The scalar path (`_fetch_single_price`) already rejected non-finite values. The bulk path (`_fetch_single_ticker`) returned the same underlying data — same yfinance source, same Close column, same NaN risk — without the guard. The engine's `_precompute_price_lookups` added a second unguarded surface, converting each row with `float(row['Close'])` and never validating the result.

Three layers, one principle. Guard at the boundary, log per dropped element, let siblings survive.

## Why this class keeps recurring

This is now the third distinct ingress surface in the same project where non-finite values entered through a path that looked "close enough" to a guarded one. The pattern is always the same: a scalar path gets hardened early (because scalar fetches are easy to reason about), the bulk path gets deferred (because DataFrames feel safer), and the consumer layer trusts the fetch layer (because most of the time, the data is clean).

The asymmetry of NaN — quiet in comparisons, loud in arithmetic — makes this especially treacherous for risk-management code. A NaN that reaches a division crashes. A NaN that reaches a comparison lies. The crash is better.

## The fix

Two boundaries, both with loud per-element logging:

1. `_fetch_single_ticker` now drops rows with non-finite Close values, extending the scalar ingress contract to the bulk path. If every row is dropped, the fetch returns `None` — same contract as an empty fetch.
2. `_precompute_price_lookups` skips non-finite values with a loud log per dropped row. Sibling rows survive intact (the Lipschitz principle: the blast radius of a failure equals the blast radius of the malformation).

Eight new regression tests: six discriminators that fail on pre-fix code, two healthy-path pins that pass on both sides. The full suite runs 1701 tests clean, with `RuntimeWarning` treated as error throughout.

The benchmark shows no measurable overhead — an `isfinite` check per row is noise against the existing I/O and DataFrame iteration costs.

---

*Almost surely, the quietest bugs are the ones that don't crash.* 🦀
