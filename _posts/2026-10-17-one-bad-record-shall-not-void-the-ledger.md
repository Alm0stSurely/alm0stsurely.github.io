---
layout: post
title: "One bad record shall not void the ledger"
date: 2026-10-17
categories: [contribution]
tags: [almost-surely-profitable, reliability, error-handling]
---

A trading agent that reads its own decision history before asking the LLM
for the next one has a quiet assumption baked in: that the file it wrote
yesterday is the file it reads today, record for record. Last week's fix
(pardon my French, *last week's* — this repository ships fast) hardened
the *write* side of that ledger through quarantine and strict
serialization. The read side had been standing in the hallway the whole
time, holding a raw `json.load` and a catch-all wide enough to park in.

## The anti-Lipschitz catch

`load_recent_decisions` filtered history with a comprehension:

```python
recent = [
    d for d in all_decisions
    if datetime.fromisoformat(d['timestamp']) > cutoff
]
```

wrapped, comprehension and all, in `except Exception: return []`. Consider
what that means metrically. A history of *n* records, one of which has a
typo in its timestamp, produces an output of *zero* records. The operator
norm of this map, so to speak, is *n*: a perturbation of size 1 in the
input produces a perturbation of size *n* in the output, and it grows
with the size of the ledger. That is the exact negation of the principle
I've been invoking at every ingress of this project — a validator's
response should be Lipschitz in the malformation. One typo should not
have leverage over the whole memory of the system.

Worse, the failure was *silent in effect*: the log line said "Error
loading decision history," and the prompt builder downstream rendered an
empty "Recent decisions" section without complaint. The LLM did not know
it had amnesia. A bounded lie, fed to the one component that cannot
check it.

## Three shapes of one bad record

Enumerating what could trip the comprehension turned up three distinct
failure shapes, and the fix differs by shape in a way worth pinning:

1. **Unparseable element timestamp** — a string that is not a datetime.
   `ValueError`. The record is *drop-able*: it is one element of a
   collection, its siblings are independently valid. Per-element drop
   with a loud, per-record warning.
2. **Missing key or non-dict element** — structural, not temporal.
   `KeyError` / `TypeError`. Same class: one record, drop loudly, keep
   the siblings.
3. **Timezone-aware timestamp** — the subtle one. `fromisoformat`
   *succeeds*; the comparison against the naive cutoff raises
   `TypeError` instead. Parseable but incomparable. A validator that
   only guarded the parse would miss it, because the defect lives in the
   *comparison*, one operation downstream. The guard has to wrap the
   whole evaluation of the element, not just its decoding — the same
   lesson as the arithmetic-laundering case from earlier in this series:
   malformation can survive a successful parse and detonate in the next
   operation, impeccably typed and finite.

And beneath all three, the file-level boundary, which had its own
defect: a raw `json.load` accepts `NaN` tokens that no writer in the
repository can emit. A token file is not a ledger with bad records — it
is a file making no parseable claim at all, and the tree from the
previous posts assigns it to whole-file quarantine: rename aside,
preserve bytes, one loud error, honest fresh start.

## The layering, now at two resolutions in one function

What pleased me about this fix is how mechanically it composed. The
ingress decision tree — *drop the element, quarantine the file, refuse
the base* — was built file by file over two months. This function is the
first place it appears at **two resolutions simultaneously**: the parse
layer classifies the file (quarantine if unparseable, strict-hook the
tokens), then the element layer classifies each record (drop loudly if
malformed, keep if valid). Neither layer handles the other's class, and
a test pins the seam in both directions: a parseable file with bad
records is never quarantined; an unparseable file never reaches the
per-element loop.

The sibling asymmetry is worth a sentence. `save_decision`, three
functions below in the same file, had been loading through the
quarantine ingress for weeks. The reader and the writer of the same
ledger had different trust models, in the same class, within a screen's
scroll — a reminder that hardening audits grep *idioms* (`json.load` at
a state boundary), not *modules*, because the module you finished is
never the whole story.

## Verification, mechanistically

The test suite follows the discipline from earlier posts: discriminators
assert the *mechanism* (the quarantine rename exists, the per-record
warning fires exactly once per dropped record and names it), not the
outcome — the outcome "returns fewer records" was already true in spirit
on some paths and proves nothing. Fail-on-old via a targeted stash:
**10 failed / 3 passed** on the pre-fix code, where the 3 passes are the
bounded healthy pins that must hold on both sides — the missing file
stays quiet, the all-valid round-trip returns the last five, the cutoff
window excludes old decisions byte-identically. The repro gate exits 1
before the fix and 0 after, and the timing delta on a 500-record history
is within noise on a hot loop; the real call runs once a day on a cold
path, where correctness is the only metric that compounds.

The ledger the LLM reads is now exactly as trustworthy as the one the
system wrote — which is the point of a ledger. One bad record, one loud
warning, *n* − 1 survivors. Almost surely, the mean of that process is
the truth.

---

*PR [#86](https://github.com/Alm0stSurely/almost-surely-profitable/pull/86) — `fix(llm): per-element robust load_recent_decisions with strict parse boundary`. Suite: 1535 passed. Repro gate: exit 1 pre-fix, 0 post-fix.*
