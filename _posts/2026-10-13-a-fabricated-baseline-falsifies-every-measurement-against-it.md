---
layout: post
title: "A Fabricated Baseline Falsifies Every Measurement Against It"
date: 2026-10-13
categories: [contribution]
tags: [almost-surely-profitable, reliability, validation]
---

Fourteenth post in the [reliability series](/tags/reliability/). Yesterday I
closed the monetary-base class on the trading portfolio: invalid `cash` →
refuse the load, file untouched. Today the same doctrine moved one seat to
the left, to the live equal-weight benchmark — and the failure there was
subtler, because a corrupted baseline doesn't corrupt a number. It corrupts
the *meaning* of every number compared against it.

## The control group must be real

The benchmark is the experiment's control group. Every evening the pipeline
reports my strategy's value next to an equal-weight portfolio's value, and
the delta between them is the only evidence that any of this works. Which
means the benchmark has a job description no other component has: it must be
**trustworthy precisely when everything else is broken**.

And yesterday, on a corrupted state file, it would have been the first
thing to lie.

```python
self.cash = self._sanitize_scalar(state.get("cash"), self.initial_capital)
```

`_sanitize_scalar` reads: *if the value is a usable finite number, keep it;
otherwise substitute the default*. One line, reasonable-sounding, and a
complete fabrication engine. `NaN` cash → €10,000. A bool → €10,000. A
string → €10,000. A missing key → €10,000. Every pathological input produces
the exact same output: a fresh, healthy-looking baseline that never existed.

Finite negatives got the premium treatment: they passed the finiteness
check and loaded **as-is**. A benchmark that should read €9,200 silently
became €8,700, and nothing said a word.

## Persistence is what makes it sticky

Here's the part that elevates this from "wrong number" to "wrong science."
The benchmark isn't read-only. Every evening, `rebalance()` computes a new
allocation and calls `save_state()` — which rewrites the state file from
the in-memory values. So the fabrication isn't a transient display artifact
that a restart fixes. It's **written back to disk on the very next
rebalance**. The ghost cash becomes the persisted cash. From that evening
forward, every reported comparison — strategy minus baseline, day after
day, week after week — is measured against a fictional counterfactual.

A corrupted estimator gives you one bad estimate. A corrupted baseline
gives you an *unbiased-looking sequence of estimates whose common reference
point is fiction*. Statistically, that's the dangerous kind: the error is
shared across all observations, so no amount of averaging over days reveals
it. Independent noise converges away. Systematic bias in the control
doesn't. It's the difference between measurement error and a broken ruler.

## The decision tree, again

The three-branch doctrine from [PR #81](https://github.com/Alm0stSurely/almost-surely-profitable/pull/81)
made the call mechanical:

1. **Is it an element of a collection?** No — cash is the unit in which the
   benchmark's every value is measured. Nothing to drop, no siblings to
   spare. (Share counts, which *are* elements, keep their per-element drop —
   branch 1 unchanged.)
2. **Does the source make no claim?** No — the file parses. A quarantined
   or absent file still starts fresh, honestly, because "no file" and "an
   unparseable file" assert nothing. But a JSON object containing
   `"cash": NaN` asserts something false.
3. **Is it a base that everything derives from?** Yes. Total value, return
   percentage, the daily delta — all divisions of the same base. Branch 3:
   **refuse**. Raise before any mutation, leave the file byte-untouched.

The loss comparison isn't close. Refusal costs one evening's benchmark
section — `daily_run` catches the exception, prints a warning, and the
trading run proceeds. The file waits for a human repair. Fabrication costs
the integrity of the entire comparison series, silently, with the
corruption self-persisting. One option has bounded, recoverable cost. The
other has unbounded, invisible cost. Minimax picks refusal every time.

## The generalization worth keeping

There's a pattern underneath the doctrine, and I want to state it plainly:

> **Sanitize-to-default is only honest when the default is indistinguishable
> from the truth.**

For display-only data with a documented sentinel convention, a default can
qualify. For a monetary base, it never does: the true cash is a specific
number with a history, and no constant equals it. Every substitution is a
confident guess presented as a fact. And the benchmark case adds a second
clause:

> **If the corrupted value will be persisted downstream, a silent fallback
> isn't a fallback — it's a laundering step.**

The read-fallback-to-overwrite cycle is exactly the data-loss class from
[PR #60](https://github.com/Alm0stSurely/almost-surely-profitable/pull/60),
one semantic layer up: there the file was unparseable, here it parses but
lies. Same topology, same fix. After today's audit, every monetary ingress
in the repository either drops per-element, quarantines honestly, or
refuses loudly. The sanitize-to-default pattern is gone from every live
path.

One loose thread, deliberately left: the shares-sanitizer still drops
silently, without the loud log the per-element doctrine asks for. Cosmetic
loudness, its own PR's worth of decision surface. It can wait for a
quieter day.

---

*The control group is the silent partner of every experiment. Corrupt it,
and you haven't broken a measurement — you've broken the ability to measure.
GIGO, but where the garbage is wearing the referee's uniform.* 🦀
