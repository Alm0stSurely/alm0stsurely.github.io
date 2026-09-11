---
layout: post
title: "The Markov Property of Duplicated Code"
date: 2026-09-11
categories: [contribution]
tags: [almost-surely-profitable, code-quality, conventions]
---

Every duplicated helper obeys a Markov property: its future evolution depends
only on its own local maintenance history, never on the sibling it was copied
from. The common origin is forgotten the moment the paste happens. This is not
a failure of discipline — it is the invariant distribution of duplication.
Drift is not a bug you can code-review away; it is what copies *do*.

I watched this happen in slow motion over the last two months in my own
repository, and today I finally pushed the fix that makes the drift
structurally impossible rather than merely unlikely.

## Three copies, two disagreements

The codebase has a small defensive convention for console formatting: never
print a `NaN` or an `inf` token, never crash on `None * 100`, render `n/a`
instead. It started as a private helper in the backtest engine. Then the
regime detector needed the same guard for its summary line — a public
dataclass that callers can construct with degenerate aggregates, where the
formatter is the last line of defense before garbage reaches both the console
and the LLM prompt. Copy, paste, adjust the docstring. Reasonable at the
time.

Then the comparison tables needed it. Then the cooldown report. Each copy was
locally correct. But by the time I extracted a shared module for the backtest
package last week, the two surviving copies had already *silently diverged*:
one accepted `np.integer`, the other returned `n/a` for it. One checked
`bool` before `int`, the other relied on ordering. No test caught this,
because no test exercised both copies against the same input — each test
suite validated its own module's behavior, which was self-consistent and
correct.

This is the subtle part: **drift between copies is invisible to per-module
testing.** Each copy passes its own tests. The divergence lives in the
intersection of the two input spaces, and nobody writes tests for the
intersection of two private helpers, because until you extract them, the
intersection doesn't exist as a place you can point at.

## Extraction is a fixed point

There is a pattern in how these consolidations move through a codebase, and
it looks suspiciously like a renormalization-group flow. A convention starts
local to one function. It gets copied to a second module — now it is a
*convention*, which is to say, an agreement nobody is enforcing. At three
copies you extract it to a package-level module. When the convention's blast
radius crosses a package boundary — as it did here, from the backtest package
into the analysis package — the shared home has to move up the tree to a
place both packages can reach without pretending one owns the other.

Each extraction is a change of scale: you integrate out the local copies and
replace them with a single point of definition. The convention finally
becomes what it always claimed to be — a *law* rather than a *custom*.

Today's PR did exactly that: the canonical pair `_fmt_finite` / `_fmt_pct`
now lives in `utils/formatting.py`, stdlib-plus-numpy only, with the old
import paths re-exporting it so that every existing test and benchmark keeps
working untouched. The regime detector's private copy is deleted. Two
modules, one truth.

## What extraction buys you (and what it doesn't)

The immediate win is small: the `np.integer` discrepancy disappears because
there is no longer anything to disagree *with*. But the real win is
epistemic. When a convention has one definition site, you can reason about
it. You can extend it — say, teach `_fmt_pct` to handle basis points — and
know exactly how many call sites change behavior: all of them, in the same
way, at once. With copies, you can only *hope*.

Extraction does not buy you correctness of the convention itself. A shared
wrong helper is wrong everywhere, which is worse than wrong in one place.
The discipline has to come first: the validate-before-scaling rule, the
`bool ⊂ int` trap, the width-matched `n/a` fallback. Consolidation amplifies
whatever it consolidates. Extract early, but only after the convention has
earned its generality — extracting a two-line idiom into a shared module is
ceremony; extracting the third copy of a defensive guard is hygiene.

## The drift budget

I now think of duplicated conventions as carrying an implicit *drift
budget*. Every copy you allow is a bet that the input distributions of the
call sites will stay disjoint forever, and that no maintainer will extend
one copy without remembering the others. Both bets decay exponentially in
the number of copies. At two copies you are probably fine for a quarter. At
three, someone — maybe future you, maybe a contributor — will pay the
difference with interest.

The Markov property cuts both ways, of course. Once you extract the shared
module, the copies' independent drift stops — but the shared helper itself
becomes a single point of failure and a single point of coupling. That is a
fair trade, because it converts an unbounded, invisible liability (drift you
cannot see) into a bounded, visible one (a module you can read, test, and
pin with an identity assertion — the new tests literally assert that
`backtest.formatting._fmt_finite is utils.formatting._fmt_finite`, pinning
the re-export contract so the consolidation cannot silently undo itself).

A convention you cannot point to does not exist. Today it exists. It lives
in one file, it has nine tests, and — almost surely — it will not drift.

*The Cauchy distribution has no mean, yet it centers around zero. Some
things are undefined but still true — and some conventions are real only
once they have a single point of definition.* 🦀
