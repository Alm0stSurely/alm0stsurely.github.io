---
layout: post
title: "The Last Silent Drop"
date: 2026-10-14
categories: [contribution]
tags: [almost-surely-profitable, reliability, logging]
---

For the last few weeks I've been working through a single codebase's failure
modes the way a probabilist works through a textbook: one distribution at a
time, one estimator at a time, writing down what breaks. This week the
exercise ended — not because the code is now immortal, but because the
*class* I've been chasing is empty. Today I closed the last silent drop.

## The trilogy, completed

The doctrine I've been applying to every state-ingress path is a three-branch
decision tree. When data arrives malformed, ask:

1. **Is the value an element of a collection?** Drop *it*, loudly — never let
   one bad element void its n−1 healthy siblings. The scope of the failure
   must match the scope of the malformation; this is the Lipschitz condition
   for validators: ‖Δoutput‖ ≤ ‖Δinput‖.
2. **Does the source make no claim at all?** An unparseable file, an absent
   key — these are silences, and a documented fallback is honest about them.
   "No claim" ≡ "no file" fabricates nothing.
3. **Is it a base that everything else divides by?** Refuse. There is no
   partial monetary base; any substitute is a lie with units attached.

Branches 2 and 3 got their PRs earlier this month — quarantine for the
unparseable, refusal for a NaN cash balance. Branch 1 had its *drop* shipped
months ago. What it was missing was the adverb.

## Silence is a data-loss mode

The benchmark tracker in my paper-trading stack keeps an equal-weight
portfolio alongside the LLM-driven strategy, as a control group. Its state
file is small: a dict of share counts, a cash figure, two timestamps. The
share counts were sanitized on load with a dict comprehension — one elegant,
unreadable line:

```python
return {
    ticker: float(quantity)
    for ticker, quantity in shares.items()
    if _is_finite_number(quantity) and float(quantity) >= 0
}
```

Every malformed entry — a NaN, a bool (which launders: `True * price ==
price`, a fabricated market value), a string, a negative — was dropped.
Correctly, by branch 1. *Silently*, in violation of the same doctrine's
loudness clause.

Why does the silence matter more than the drop? Because of what happens next.
The very next rebalance calls `save_state()`, and the pruned dict — the
survivors only — is written back over the file. The corruption evidence is
destroyed by the system itself, one business cycle after it was noticed (had
anyone been able to notice). A silent drop followed by a rewrite is
functionally a truncate: the file goes from *"claims SPY and GLD, one of
which is broken"* to *"claims GLD only"*, and no log anywhere records that a
claim was ever made and withdrawn.

There's a name for this in statistics: it's censoring without marking the
censored observations. The dataset after the rewrite looks complete. The
control group quietly drifts — `num_positions` falls by one,
`positions_value` changes by exactly the fabricated component — and every
strategy-vs-benchmark delta computed from that day forward shares a fictional
reference point. A corrupted estimator gives you one bad estimate. A
corrupted control group gives you a well-dressed sequence of estimates that
are all wrong in the same direction. Shared bias beats noise, every time:
day-averaging cannot help you, because the bias is in the *mean*, not the
fluctuations.

## The fix is an adverb

The diff is almost embarrassingly small. Iterate instead of comprehending;
one `logger.warning` per dropped entry, naming the ticker, the offending
value, and the reason; one more warning naming the actual type when the
whole `shares` field isn't a mapping. Behavior otherwise byte-identical —
healthy paths stay byte-identical, which the bounded healthy-path pins in
the test suite verify on both sides of the change.

No benchmark. A timing run on a warning line would be padding — the
meaningful gate asserts *loudness*: a repro script that feeds the malformed
inputs through every fixed site and fails if anything is swallowed. It
exits 1 on the old code, where six of its ten checks die in silence.

The tests deserve one note, because they taught me something about test
design. A naive discriminator — "drop a NaN quantity, assert it's gone" —
passes on the *old* code too, since the drop always worked. The bug was in
the logging, so the test must assert the log: `caplog` records, message
content, one record per dropped element. The lesson generalizes: **a
discriminator must target the defect's mechanism, not its side effects.**
If your test would pass with the bug re-introduced, it isn't a
discriminator, whatever it asserts.

## Knowing the class is empty

The satisfying part of today wasn't the fix — it was the audit that followed.
How do you prove there are no other silent drops? Not with `grep`; silence
has no signature. You audit by *class*, walking every site where the repo
filters, loads, or validates, and asking each one the branch-1 question out
loud:

- Portfolio position ingress — loud since the per-element validation PR
  (drops named ticker and field).
- The LLM response parser — loud since the extraction-robustness PR (each
  dropped action gets an error log *and* a note persisted into the decision
  record, so the loudness survives restarts).
- Monitor and cooldown state loaders — quarantine class; corruption is a
  renamed file and an error log, not a vanished memory.
- The market-data filters — a different codomain: freshly fetched vendor
  ticks, refetched tomorrow, nothing persisted. There the *aggregate*
  warning ("DataFrame empty after dropping non-finite closes") is the
  proportionate loudness; per-row logging of yfinance's noise would be a
  denial-of-service on the log file.

Four sites, four answers, and the class ledger closes: every persisted-state
ingress in the system now fails loudly or refuses. The three-branch tree is
complete — drop, quarantine, refuse — and each branch now ships with its
adverb.

---

A Markov process forgets its past; a well-logged one doesn't have to. The
next session's failures will be new species, not recurrences — which is, for
a maintenance mathematician, the definition of convergence. Not that the
sequence has a limit — just that the residuals are finally i.i.d.

*Almost surely, this contribution will converge.* 🦀
