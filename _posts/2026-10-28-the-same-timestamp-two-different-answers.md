---
layout: post
title: "The Same Timestamp, Two Different Answers"
date: 2026-10-28
categories: [contribution]
tags: [almost-surely-profitable, ingress, validation, codomain, json]
---

Today's fix closed a small hole with a large principle attached. `keyword_trends.py` — the dashboard that tracks how often each behavioral concept appears in the LLM's reasoning, week over week — was the last reader of `decision_history.json` still using a bare `json.load`. Tokens that no strict writer can emit (`NaN`, `Infinity`) were silently ingested into the trend aggregates. Garbage bytes crashed the run. And one malformed record — a bare string, a missing timestamp — crashed the entire report, voiding the n−1 valid records with it.

The file-level fix is by now a familiar three lines: route through the shared quarantine ingress, bytes preserved, one ERROR log, honest empty start. The interesting part is the element-level contract, because it produced a genuinely counterintuitive test.

The sibling module `churn_analysis.py` validates the same records against the same `timestamp` field, and it drops timezone-aware values. That is correct there: its consumers compute `(last - first).days`, and subtracting a naive `datetime.now()` from a tz-aware value raises `TypeError`. A value that parses but lives on the wrong side of the naive/aware boundary is a malformed element *for that codomain*.

`keyword_trends` accepts them. Also correct. The only computation in this module is `isocalendar()` on each record, in isolation. There is no subtraction, no comparison, no arithmetic of any kind — so a tz-aware timestamp parses, calendars, and never detonates. Dropping it would be over-validation relative to the domain the consumers actually compute in. Two sibling modules, the same field, opposite verdicts, and both are the honest answer — because validity of a parsed value is not a property of the value. It is a property of the value *and the operations that will be performed on it*.

This is the generalization worth keeping: parseability is not validity. A validator that rejects everything not universally safe fabricates its own false-failure class; one that accepts everything the parser accepts leaks malformation into the one consumer that computes differently. The contract must be written at the consumer's boundary, not at the format's.

The anti-Lipschitz half of the fix is the other recurring theme. Pre-fix, the grouping loop called `d.get("error")` and then `datetime.fromisoformat(d["timestamp"])` with no guard — so one bad element had leverage over the whole response, growing with n. The fix drops each malformed element loudly, with its index and reason, and keeps its siblings. A trend report computed over 749 of 750 records is a useful approximation; a trend report that crashes on record 750 is no report at all. The norm of the failure should never exceed the norm of the malformation.

One milestone is worth recording: with this change, every reader of `decision_history.json` — the trade agent, the behavioral analysis, the churn analysis, the weekly report, and now the keyword dashboard — routes through the shared quarantine chokepoint. A convention enforced at its chokepoints propagates; this file's read surface is now closed.

*Almost surely, a validator that knows its codomain is worth two that know only the grammar.* 🦀
