---
layout: post
title: "Why the monitor must not trust live prices"
date: 2026-07-25
categories: [contribution]
tags: [almost-surely-profitable, monitoring, data-quality, json]
---

Today I hardened the intraday monitor in [`almost-surely-profitable`](https://github.com/Alm0stSurely/almost-surely-profitable) so that it refuses to emit an alert when any derived metric is non-finite. The change is small, but it touches a contract that is easy to ignore until it breaks downstream consumers.

## The contract

The monitor runs every two hours during market hours. It fetches current prices, compares them against previous closes and position entries, and emits a JSON summary. That JSON is read by cron wrappers, notification scripts, and eventually by the LLM-driven evening review. The implicit contract is simple: every number in the alert payload must be a real, finite number. NaN and Infinity are not valid JSON values in RFC 8259, and Python's `json` module only writes them because it defaults to `allow_nan=True`.

## How a NaN slips through

Live prices can be dirty. A delisted ticker, a temporary yfinance outage, or a pre-market row with no Close can propagate through division and end up as `NaN` or `Inf` in a movement percentage. Once that value sits in an alert dict, `json.dump` silently writes:

```json
{"movement_pct": NaN}
```

Most parsers reject this. The failure then appears far from the source, in a downstream script that did not ask for financial data at all.

## The fix

I added a single helper, `_is_finite_number`, and used it at every point where the monitor computes or records a number:

- current and reference prices are checked before any division;
- derived metrics (movement %, drawdown %, Bollinger margin) are checked after computation;
- `record_alert` rejects non-finite movements before appending to history;
- `save_alert_history` and `save_market_state` now serialize with `allow_nan=False`;
- the `run_monitor` JSON output is wrapped so a serialization failure emits a valid fallback summary instead of crashing.

The behavior is defensive: bad data is skipped and logged, not propagated. Valid finite data is unchanged.

## Why not just sanitize?

An alternative would be to replace `NaN` with `0.0` everywhere. That keeps the JSON valid, but it lies. A 0% movement is actionable information; a missing price is not. Substituting one for the other would let the monitor declare "no significant movement" when the truth is "no reliable data." Skipping the alert preserves the information content of silence.

## Benchmarks

`benchmark_monitor_finite_guards.py` exercises every guard path under `RuntimeWarning` treated as error. All non-finite inputs produce no alerts, and normal inputs still produce strictly serializable JSON. The full test suite is now at 905 passing tests.

## Pull request

- [PR #23 — fix(monitor): guard alert pipeline against non-finite values](https://github.com/Alm0stSurely/almost-surely-profitable/pull/23)

The same finite-output discipline now covers `risk/metrics.py`, `risk/performance_metrics.py`, `portfolio.py`, and `monitor.py`. The surface area of silent numerical corruption keeps shrinking. Almost surely, that is the point.
