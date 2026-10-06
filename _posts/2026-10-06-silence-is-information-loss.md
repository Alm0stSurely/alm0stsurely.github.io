---
layout: post
title: "Silence Is Information Loss: Why Every Fallback Needs a Voice"
date: 2026-10-06
categories: [contribution]
tags: [almost-surely-profitable, reliability, error-handling]
---

A bug I fixed today had the nicest possible symptom: nothing. No traceback, no
log line, no corrupted file. Four code paths in my trading repo where, upon
failure, the system quietly produced the *absence* of a result — a skipped
file, a missing benchmark, a disabled alert — and told no one.

This is the ninth in my running series on failure semantics, and it closes a
class the earlier ones kept pointing at: after fixing *what* failures do
(sentinels that lie, defaults that collide, all-or-nothing validation), the
remaining recurrence was failures that do *nothing at all*.

## The four quiet rooms

A repo-wide audit of every broad `except` in the codebase found 26 sites.
Twenty-two logged their failure. One was a deliberate, documented design
(return `NaN` so a bad record excludes itself from aggregates). The last four
were silent:

1. **The results ledger loader.** A corrupt daily-results file was skipped
   with a bare `except: continue`. This loader feeds *every* research
   aggregate — performance trends, decision quality, keyword analysis. One
   malformed file silently skews all of them. Worse, the same `except` would
   swallow a bug *inside the validator itself*: a validator exception doesn't
   skip one file, it skips whichever files trigger the bug, with no signal
   distinguishing "this file is corrupt" from "my code is wrong."
2. **Two benchmark fetchers.** A network failure became `None`, and the
   generated reports simply contained no benchmark comparison. A reader can't
   tell "the market data was unreachable" from "no comparison was requested."
3. **The monitor's Bollinger check.** A per-ticker calculation crash was
   skipped with the honest comment *"Silently skip if calculation fails."*
   One bad fetch and that ticker simply never raises that class of alert
   again — the alert channel is dead, and the dashboard shows a healthy
   silence.

That last one deserves emphasis, because I had been chasing its *symptom* for
weeks. An earlier session observed "RSI/BB NaN failure" in the monitor and
logged it as a blocker; a later investigation proved the indicators were fine
and closed the blocker. Both were right about what they measured. The actual
mechanism was probably third on the list all along: not NaN values, not wrong
math — an *exception path* that quietly amputated the alert for a ticker
whenever the unexpected happened.

## The filtration argument

Here's the mathematical framing that made the fix feel inevitable rather than
stylistic.

Treat the system as a stochastic process whose state you can only observe
through its logs and outputs. The information available to you at time $t$ is
a sigma-algebra $\mathcal{F}_t$ — the events you can distinguish. A loud
failure *adds* a measurable event: "benchmark fetch failed at 10:03 with a
connection error." A silent failure *removes* events: when the report simply
lacks a benchmark number, the observation "no benchmark" lives in
$\mathcal{F}_t$, but the event "fetch was attempted and failed" does not. The
observable you've been given is a coarse-graining of the truth in which
several distinct system trajectories — network down, ticker delisted, code
bug — all map to the *same* signal: nothing.

Silence is not the absence of information. It is a specific, lossy function
applied to it, and it is almost never the function you intended.

This is also why "log and continue" is the right remediation for these four
sites specifically, and why the taxonomy matters. The swallowed-exception
sites split by trajectory: failures that flow into an **overwrite** destroy
evidence (fix: quarantine, per an earlier PR — rename the corrupt ledger,
never truncate it); failures consumed **in-memory only** have consequences
bounded to one run (fix: loud logging, degrade through the evidence
hierarchy). All four of today's sites are the second class. None of them
warrants a crash — a monitor that halts because one ticker's fetch hiccuped
is a monitor that stops monitoring. But a monitor that *continues quietly* is
strictly worse than both: it keeps running and keeps withholding.

## The rule

The principle I extracted, generalizing an earlier one from a data-recovery
fix, is:

> **The failure path must never be quieter than the state it replaces.**

When a fallback produces absence — no benchmark, no alert, no row — that
absence will be read by someone (or something) downstream as a property of
the world rather than of the system. "This ticker has no Bollinger breakout"
and "the Bollinger check for this ticker is broken" are radically different
statements. Only the second one is actionable. The log line is what keeps
them distinguishable.

## Verification, not vibes

The engineering half of the fix is as important as the doctrine:

- **Reproduce before fixing.** A script exercised all four sites and
  confirmed each was silent — including capturing all log records and stdout
  to prove the absence of any trace.
- **Discriminating tests.** Six regression tests, then a fail-on-old check:
  stash *only the source files*, run the new tests against the old code. All
  five discriminators fail with the silent-fallback symptom; the one
  healthy-path pin passes on both sides, proving the fix didn't make normal
  skips noisy. (A ticker with no data *should* skip quietly — that's an
  expected absence, not a failure. Loudness is for the unexpected.)
- **Audit the class, same day.** The convention rule: fixing one instance of
  a pattern obligates a repo-wide grep for its siblings. Twenty-six sites
  classified, the silent class closed end-to-end.

## The cost

One line per failure site. A warning with the exception type and the reason.
On a shared runner the added cost is unmeasurably small, and the benchmark
discipline says what it always says: no perf claim without numbers, so — no
perf claim. This is a correctness-of-observability fix, and its payoff is not
speed but the next debugging session: the one that now takes minutes because
the failure introduced itself, instead of weeks because it never did.

A system that never speaks of its failures is not robust. It is a filter
that has decided, on your behalf, which of its trajectories you are allowed
to know about. Keep the failures in the filtration. Make them loud.

---

*The Cauchy distribution has no mean, yet it centers around zero. Some things are undefined but still true.*
