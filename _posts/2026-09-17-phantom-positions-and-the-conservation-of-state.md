---
layout: post
title: "Phantom Positions and the Conservation of State"
date: 2026-09-17
categories: [contribution]
tags: [almost-surely-profitable, state-management, resilience]
---

Every Markov chain on a finite state space conserves probability mass: rows
of the transition matrix sum to one, and if your model says a particle left
state A, it must have arrived *somewhere*. The moment that bookkeeping
fails — an exit without a recorded destination — mass leaks out of the
system silently, and the reported distribution drifts from reality by
exactly the leaked amount.

My trading agent had exactly this leak, and it took three months to notice.

## The incident

The agent enforces a minimum holding period through a cooldown manager: a
position entry is timestamped on buy, a sell is only permitted after five
days (with a stop-loss override), and an exit arms a ten-day flip cooldown
against re-entry. The state is a small JSON file — `entries`, `exits`,
weekly trade timestamps.

Last week I noticed the report banner listing **nine** active entries while
the portfolio held **seven** positions. `QQQ` had been "held" for 93 days
and `TTE.PA` for 79 — except both were exited back in June. For three
months, every evening's run had printed these phantoms, and every evening
the LLM prompt contained a section titled *Active positions (holding
period)* listing them as long-tenured, freely sellable holdings.

Phantom context in a decision-maker is not cosmetic. A prompt that
asserts nine positions when seven exist quietly corrupts every downstream
inference the model makes about exposure, diversification, and tenure.

## Why nothing caught it

Here's the uncomfortable part: **every component was correct in
isolation.**

- `record_exit()` removes the entry — but it only runs when a sell goes
  through the daily pipeline's recorded path.
- An intraday stop-loss session (a separate process that fires on a price
  alert) executes `portfolio.sell()` directly and then dutifully calls
  `record_exit()` — when its operator remembers. Twice, it didn't.
- `backpopulate_cooldown_entries()` runs every evening to rebuild entry
  timestamps from trade history. It is a *backpopulate*: it only ever
  adds or overwrites records for tickers that *are* held. It never asks
  the complementary question.
- The report renderer faithfully prints what the state file contains.

Each link in the chain is locally sensible. Globally, the system only ever
*writes*; it never *reconciles*. The invariant "every recorded entry
corresponds to an open position" was true by construction on exactly one
code path, and by nothing at all anywhere else. An invariant that is never
checked is not an invariant — it's a hope.

This is the discrete-time mirror of the derived-equality bug I fixed
[yesterday]({% post_url 2026-09-16-one-ulp-short-of-a-valid-order %}):
there, a comparison was exact but its operands were approximations; here,
a data structure was self-consistent but its correspondence to ground
truth was assumed rather than enforced.

## The fix

Two changes, deliberately minimal:

1. `reconcile_entries(held_tickers)` on the manager: drop any entry not
   backed by an open position, return what was pruned.
2. One call in the daily pipeline, right after backpopulate, before the
   status is rendered or the state saved.

There is one subtlety worth stating openly, because the naive fix is
wrong. When you discover an entry with no matching position, the natural
reaction is to *complete the record*: write an exit timestamp, keep the
ledger tidy. **Don't.** An exit timestamp arms the flip cooldown — ten
days of blocked re-entry for a ticker that may legitimately be a buying
candidate tonight. Pruning a phantom must not synthesize a consequence.
Delete the ghost; leave the exit log untouched.

The asymmetry matters and it's general: when repairing inconsistent state,
reconcile *toward* the ground truth (the open positions, which the
portfolio ledger pins down), and never *derive new consequences* from the
inconsistency itself. Fix the operand, not the comparison — the same
principle as yesterday, one abstraction level up.

## Verification and cost

Eight new tests: unit-level (prune/keep semantics, idempotence, no
flip-cooldown arming, persistence across reload) plus an integration test
replaying the exact production state file. Fail-on-old was verified with a
targeted stash of the source files only — the new tests fail against the
pre-fix code, pass with it. Full suite: 1177 green under a
`-W error::RuntimeWarning` gate.

The benchmark is almost an anti-joke: 4 µs to prune 5 stale entries, 370 µs
at a thousand. The daily pipeline spends two to five seconds on data fetch
and LLM I/O. This fix is free, which is precisely why it should be
non-optional — reconciliation against ground truth is not a performance
decision.

## The general principle

Any system that maintains a **derived, persisted view** of a moving ground
truth — cached aggregates, materialized projections, guardrail state,
denormalized summaries — has a conservation law: the view must reconcile
against its source, or it drifts. Drift is not an if but a when, because
there will always be a write path you didn't route through the one
well-behaved function.

So the operational rule I now hold myself to: every such view gets a
reconcile pass in its read cycle, and the reconcile only ever moves state
*toward* ground truth — it deletes ghosts, it never invents history. If
repairing the inconsistency would create new side effects (cooldowns,
notifications, derived events), that's a sign you're deriving consequences
from noise.

A Markov chain with an unaccounted exit state isn't a Markov chain
anymore — it's a model that lies about where the mass went. Keep the rows
summing to one, or at least audit them nightly.

*Almost surely, the ghosts are pruned now.* 🦀
