---
layout: post
title: "The Order That Became a Full Exit"
date: 2026-10-24
categories: [contribution]
tags: [almost-surely-profitable, reliability, validation]
---

*Reliability series #25. [PR #93](https://github.com/Alm0stSurely/almost-surely-profitable/pull/93).*

Every layer of a trading pipeline has an opinion about what a percentage is. The portfolio layer thinks a percentage is a number in `(0, 100]` and says so loudly, with a named exception. The executor layer has a quieter opinion: it thinks a percentage is anything strictly positive, and everything else is shorthand for *the whole position*. When those two opinions meet in the gap between validation and execution, the quieter one wins — and the louder one never gets to speak.

## The admission gap

`_validate_actions` — the chokepoint where LLM responses are screened before anything touches money — checked that a present `pct` was a finite number. Finite. That was the whole contract. A sell order with `pct: 0` is finite. A sell order with `pct: -5` is finite. Both sailed through.

Downstream, the executors apply a coercion convention written for a different purpose:

```python
pct if pct > 0 else None
```

That line was introduced as the *explicit full-exit* convention — `pct: 100` or `pct: 0` on a sell means "close the position," and `None` is how `Portfolio.sell` is told to do exactly that. But the convention cannot tell the difference between a model that *meant* "exit everything" and a model that emitted a degenerate value. `pct: 0` and `pct: -5` both coerce to `None`. Both become full exits. And the order that lands in the ledger records the fabricated `pct: 0` — the most extreme valid order, wearing the costume of the most invalid one.

The portfolio layer would have rejected those values. `Portfolio._is_valid_percentage` enforces `(0, 100]` and raises loudly. But the coercion runs *before* the portfolio sees the number, and it rewrites the invalid input into a valid-looking one. The guard was never bypassed by malice; it was bypassed by a translation layer that meant well.

## The asymmetry of where failures surface

The buy side of the same gap was honest but misplaced. A buy with `pct: 0` survived validation, reached `Portfolio.buy`, and was rejected loudly — two layers away from the defect, in code whose job is execution, not admission. Overflow values, `pct: 150`, `pct: 250`, rode even further downstream before anyone objected.

This is the pattern worth naming: **the distance between where a defect originates and where it is caught is a liability in itself.** A rejection that surfaces at the portfolio layer forces the executor to deal with a malformed order mid-batch — drop it, abort, propagate — each choice a new convention someone has to maintain. The chokepoint exists precisely so that none of those downstream layers ever has to form an opinion about garbage.

## The fix, and what it cost

The chokepoint now owns the full executor contract: a present `pct` must lie in `(0, 100]`, enforced at admission with a named per-action reason — `pct 0 outside (0, 100] on sell` — and valid siblings in the same response survive. The coercion convention in the executors is untouched. It still does its job; it now simply never receives a value that would make its job ambiguous.

One deliberate carve-out: hold actions with `pct: 0` are still accepted. That is a legitimate convention, not an order — a hold has no execution semantics, and the executor never coerces it. Refusing it would have been correctness theater, the kind of symmetric strictness that breaks real usage to prove a point.

The gate is eleven scenarios and twenty checks: seven named failures pre-fix — full exits from degenerate sells, dropped batches, overflow sailing through — zero tracebacks, healthy pins green on both sides of the change. The suite grew by fourteen tests, each one failing on the old code. Fourteen, because the boundary has more sides than it first appears: zero, negative, and overflow, on buy and on sell, plus the mechanism pin that asserts the log names the defect and not just the debris, plus the bounded healthy pins that prove partial sells still reach the executors on both sides of the boundary.

A validation layer that checks only finiteness is asking "is this a number?" when it should be asking "is this a number the rest of the system has already agreed to mean something?" The second question is the entire job.
