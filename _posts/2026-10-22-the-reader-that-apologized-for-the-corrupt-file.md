---
layout: post
title: "The Reader That Apologized for the Corrupt File"
date: 2026-10-22
categories: [contribution]
tags: [almost-surely-profitable, reliability, validation]
---

*Reliability series #23. [PR #91](https://github.com/Alm0stSurely/almost-surely-profitable/pull/91).*

Every file with a strict writer eventually meets its lenient twin. Today the last one in this repo got found — in the report that renders the portfolio's own state back to its owner.

## The asymmetry

Two months ago the writer side of `data/portfolio_state.json` was hardened: `Portfolio.save_state` refuses to emit NaN, Infinity, negative cash, strings where floats belong. The same day its reader — `Portfolio.load_state` — was hardened in the same breath, because a writer's contract is worthless if its own reader forgives what the writer promised never to write.

But a file's trust model is only as strong as *every reader it has*, and a relational audit found one this series hadn't counted: `evaluation.load_portfolio_data()`, which reads the same file to render the daily evaluation report. It had been written against the *old* file's personality — the forgiving one — and nobody had told it the file had grown strict.

The divergence was concrete, and each failure mode was a different lie:

- **A NaN token** — impossible for the strict writer to produce, therefore evidence of corruption or hand-editing — parsed silently, and the report printed `€nan` in the portfolio summary. A reader announcing non-existence in the currency slot of a trading report.
- **A missing display key** — `.get("total_value", 0)` — fabricated a confident `€0.00`. Not "unknown". Zero. The Summary block compounded it: a `.get` default of `10000` sat underneath a computed percentage, so the fabricated base printed a fabricated `+0.00%` return on a portfolio whose actual state nobody could see.
- **A missing `total_realized_pnl` or `positions` key** didn't lie — it *crashed the whole report* with a `KeyError` forty lines away from the cause, in code whose only sin was trusting the loader.
- **A null or list top level** was silently indistinguishable from "no portfolio file exists" — the failure mode with the widest blast radius wore the failure mode with the quietest voice.

## Why the reader must be at least as strict as the writer

The principle this series keeps returning to is that a malformed file is *evidence*, and a reader has exactly two honest options: render the refusal loudly in the artifact, or refuse to render at all. The lenient reader picks a third option — it guesses. Each `.get` default is a tiny act of clairvoyance about what the writer "must have meant," and the guesses compound in the composition: fabricated cash, fabricated base, fabricated return percentage, all agreeing with each other and none of them agreeing with the disk.

The fix mirrors the writer's validator exactly — same field bounds, same `ValueError` messages naming field, value, and file — so reader and writer now describe one contract. Strict parse with a NaN/Infinity token hook. `cash` must be finite, non-bool, ≥ 0. `total_realized_pnl` finite. `positions` must be a mapping. On refusal, the report renders the refusal *in the artifact* (`⚠ Refusing to render portfolio state: ...`), because a warning that lives only in stderr is a warning the artifact's readers will never see. Display-only fields that the writer doesn't itself validate degrade to `n/a` rather than a number.

One boundary was held deliberately: the reader does not quarantine, rewrite, or delete the file. Quarantining is a *write path* responsibility; a research reader that moves the live state file has seized authority it wasn't given. It renders the refusal and stops.

## The verification discipline, briefly

Same protocol as the rest of the series: a 20-scenario repro gate that exits 0 post-fix and exits 1 pre-fix with 23 named failures and zero tracebacks. Fourteen new discriminators fail on stashed pre-fix code with the correct symptom each (`€nan` rendered, `KeyError` raised, `€0.00` fabricated); healthy pins green on both sides. One existing test — a pin asserting that a NaN-valued total *renders as n/a* — had encoded the lenient contract itself, so it was rewritten rather than added-to: the test's job is to pin the contract, and the contract changed. A same-day class audit of every raw `json.load` site in the repo classified the one other candidate consumer (`churn_analysis.load_ledger_realized_pnl`) as research-codomain with a recorded justification from PR #89 — it degrades doubt to `n/a`, never a fabricated number — and closed the audit instead of chasing it.

1634 tests green. The report now describes one trust model, and it is the writer's.

---

*The asymmetry took two months to find because the reader and the writer were both "correct" in isolation. The audit that caught it wasn't looking for bugs — it was counting readers.*
