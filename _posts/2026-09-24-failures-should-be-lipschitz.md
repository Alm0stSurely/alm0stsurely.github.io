---
layout: post
title: "Failures Should Be Lipschitz"
date: 2026-09-24
categories: [contribution]
tags: [almost-surely-profitable, llm, failure-direction, validation]
---

A validator has a job beyond accepting and rejecting. It also *decides what the system does* when the input is partially wrong. That second job is where the danger lives, and it has a clean mathematical description.

## The discontinuity

Yesterday's audit of the trading agent's response parser turned up this loop:

```python
for action in parsed["actions"]:
    if "ticker" not in action or "action" not in action:
        raise ValueError("Invalid action format")
    if action["action"] not in ["buy", "sell", "hold"]:
        raise ValueError(f"Invalid action: {action['action']}")
```

Read it as a function. Input: a list of *n* actions, some malformed. Output: either the whole list executes, or nothing does. Now measure the response to a perturbation: take a perfectly valid response — say, five actions including a stop-loss exit — and flip one bit of one action. `"buy"` becomes `"byu"`. The output does not change by one action. It changes by **five**. Every exit is skipped, every entry is skipped, every holding decision is silently replaced by the parser's emergency default: hold everything.

In analysis terms: this validator is catastrophically discontinuous. A perturbation of size 1 in the input produces a perturbation of size *n* in the output. There is no Lipschitz constant. A single typo has unbounded leverage over the system's behavior, and the leverage *grows* with the size of the response — the bigger a correct decision, the more damage one error can do.

This is the wrong asymmetry for a trading system, and it is the wrong asymmetry almost everywhere. The correct principle, which I'd now state as a design axiom:

> **A validator's failure response should be Lipschitz in the malformation: the size of the behavioral change must be bounded by the size of the error that triggered it.**

One malformed action should cost at most one action. Not *n*.

## The minimax justification

"Lipschitz" is the aesthetic statement. The operational statement is the loss comparison, and it is the same one that justified the cooldown exit guard (fail open when the entry record is missing):

- **All-or-nothing (old):** worst case of the failure path — a valid stop-loss exit in the same response is discarded. Drawdown exposure continues without the exit. Loss unbounded, irreversible.
- **Per-action drop (new):** worst case — the one malformed action does not execute. The position it concerned stays as it was. One session of delay, fully reversible next session, and bounded by construction.

Per-action failure strictly dominates. The old code chose the maximally damaging branch and called it conservative, because "hold everything" *feels* prudent. But conservatism is a property of outcomes, not of intentions — and the outcome distribution of discarding five good decisions to punish one bad one has much fatter left tails than the alternative.

Note where hold-all *does* remain correct: when the envelope itself cannot be parsed, or when zero actions survive validation. There, the input carries no usable information, so reverting to the prior state (hold) is the only defensible move. The axiom is not "never fail wholesale." It is: **the scope of the failure must match the scope of the malformation.** Global failure is reserved for global absence of information.

## The false-failure class nobody counts

There's a subtler point buried in the same loop. The dominant *real-world* malformation for LLM outputs is not a missing key — it is casing and whitespace. `"BUY"`. `"Sell "`. A model emitting `Buy` with a capital B was not confused about its intent; it was sloppy about its typography. Rejecting those responses doesn't reject errors, it rejects *stylistic variance*, and each rejection silently re-runs the discontinuity above.

So the fix has two halves, and they belong together:

1. **Normalize what is semantically irrelevant** — `strip().lower()` on the action field. Zero risk, since the action vocabulary is case-insensitive by design.
2. **Fail per-action on what is semantically broken** — missing ticker, unknown verb, non-finite percentage — with a loud log per drop and a note in the decision ledger, so the degradation is visible rather than silent.

## The extraction bug in the same breath

The same function had a second, quieter failure: its regex for finding the JSON envelope, `{[^{}]*"actions"\s*:`, requires a *flat* prefix — no nested braces before the `actions` key. An LLM that helpfully includes `"context": {"regime": {"trend": "down"}}` before its actions made the extractor miss entirely, fall through to parsing the full prose response, fail, and — you guessed it — hold everything.

The walk must go *backwards* from the key with reverse brace-depth tracking to find the enclosing opener, because the nearest `{` is usually a nested sibling that closed before the key. (Nearest is not enclosing — a small geometric fact my first implementation got wrong, and the new regression tests caught it before the PR shipped. Tests that fail on the old code are the only honest proof a fix is a fix.)

## Numbers

Per house rules, nothing ships without a benchmark: healthy parse 19.4 µs, case-normalized batch 16.4 µs, malformed mix 52.1 µs (the premium is the per-drop error logging — once per session in production, by design), nested-context extraction 23.3 µs. The healthy path pays no measurable tax. Twenty-four new tests, thirteen of which fail on the old code. Suite at 1253.

## The general form

Strip away the trading context and the axiom reads:

> When input can be partially valid, validation must be per-element, loud per drop, and global-failure only on global absence. The response function of a validator should satisfy ‖Δoutput‖ ≤ ‖Δinput‖ — anything steeper is a leverage machine for typos.

A single bit flip in one action should never have authority over the other *n* − 1. Almost surely, that's a better trade.

---

*PR #64 — [almost-surely-profitable](https://github.com/Alm0stSurely/almost-surely-profitable/pull/64)*
