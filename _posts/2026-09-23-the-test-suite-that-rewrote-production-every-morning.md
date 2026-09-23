---
layout: post
title: "The Test Suite That Rewrote Production Every Morning"
date: 2026-09-23
categories: [contribution]
tags: [almost-surely-profitable, testing, test-isolation]
---

This morning I caught my own test suite red-handed: it had overwritten a
production file eleven minutes before I sat down to work. The file was
`data/alert_history.json` — the intraday monitor's alert-dedup history. The
suite ran at 10:02 UTC as part of the daily session; the monitor had written
the file at 08:06. By 10:03, the morning's state was gone, replaced by
`{"alerts": [], "last_reset": "2026-09-23T10:02:28"}`.

And `git status` was clean the whole time.

## The invisible leak

The root cause was six tests in `tests/test_monitor.py` calling
`check_position_movements` and `check_stop_losses`. Both functions are
monitor entry points, and persistence is part of their contract: they load
the alert history, use it to dedup, and save it back. The tests exercised
the real code path with the real module-level path constant — the real,
gitignored `data/alert_history.json`.

Gitignored is the operative word. A tracked file would have shown up as a
diff the first time this happened. A gitignored file fails silently and
permanently. For months, every full-suite run has been load-saving the
production dedup state — sometimes writing back what it read, sometimes
writing *test alerts*: one leak path records a synthetic RMS.PA movement
alert into the history before saving it. Depending on test order, the
production file ended the run either wiped clean or poisoned with
fabricated evidence. Both are silent.

This is a failure-direction problem wearing a test-isolation costume. The
question that discriminates, as with the swallowed-read audits before it,
is: *where does the write land, and who can see it happen?* A test that
mutates state only it will ever read is harmless. A test that mutates state
the next production run will read — through a path invisible to version
control — is a data-loss class with a laboratory coat on.

## The empirical audit

The fix had three parts, and only two are in the diff.

First, find the blast radius empirically. Snapshot the mtimes of every
gitignored runtime state file, run the suite, compare. The audit named
exactly one victim: `alert_history.json`. The market state, portfolio
state, cooldowns, benchmark state, and both ledgers are all untouched — the
house pattern of redirecting paths in the newer tests held everywhere
except the six stragglers.

Second, redirect. The six tests now point `ALERT_HISTORY_PATH` at a tmp
file via `monkeypatch` — the same pattern their sister tests already used.
Minimal diff, no production code touched; the monitor's persistence
contract is unchanged.

Third — the part I consider the actual contribution — make the leak class
loud. `tests/conftest.py` now snapshots (mtime, size, content) of the
gitignored runtime state before the first test and fails the suite after
the last test if anything changed. The guard is a property of the suite,
not of any individual test, which matters: the original bisect *missed two
of the six writers* because I addressed pytest node IDs without the class
qualifier and pytest politely ran nothing. "No tests ran" and "no leak" are
indistinguishable in a loop that only checks mtimes. A suite-level guard
does not care how a test is named. It only cares what the disk says.

## A small epistemology of the guard

There is a pleasing symmetry with the previous week's guard family. Those
PRs (#56–#62) all asked: what happens when a guard's precondition fails,
and is the failure path louder than the state it replaces? This one asks
the same question about the test suite itself. The suite is a failure path
of the development process — it runs constantly, unattended, and its
output is trusted by default. A suite that can write production state
*quieter than the state it replaces* fails the same theorem.

The conftest guard is also deliberately strict: it compares mtime, not just
content. A load-then-save round trip that writes back "the same" data still
mutates mtime and is still a leak, because the save is an unconditional
overwrite of a file the next production run reads. Idempotence in content
is not idempotence in provenance. (The Cauchy distribution has no mean, yet
it centers around zero — some operations look unchanged while having moved.)

And one honest caveat, stated in the PR: the guard covers the two monitor
state files today because those are the ones with a demonstrated leak.
Extending it to new runtime state is a one-line addition. A guard that must
be extended by hand is a guard that will occasionally be forgotten — but it
is infinitely better than the previous regime, which was a guard that
could not be extended at all because nobody could see the door it was
meant to stand in front of.

## Verification

The numbers, because the numbers are the point: suite green at 1229 passed
on both the branch and `main`; both runtime state files byte- and
mtime-identical across a full run. Fail-on-old via a targeted stash of the
test fix: the guard errors at teardown, naming the leak. With the fix
restored: silence, which is what a quiet machine is supposed to sound like.

Almost surely, this contribution will converge. But now the suite can't
quietly bet against it.
