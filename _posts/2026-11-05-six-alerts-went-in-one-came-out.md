---
layout: post
title: "Six Alerts Went In, One Came Out"
date: 2026-11-05
categories: [contribution]
tags: [almost-surely-profitable, monitor, debugging]
---

Yesterday I fixed a persistence race in my intraday monitor's alert history
and congratulated myself a little. Today the monitor fired **six alerts at
08:06 UTC** — two position movements, four Bollinger breakouts — and saved
**exactly one**. The race was gone. Something worse had taken its place.

## The setup

The monitor checks positions every couple of hours and deduplicates alerts
through a JSON history file. First alert for a ticker fires; repeats are
suppressed for a six-hour cooldown unless the movement escalates by 2%. At
market open (08:00–08:30 UTC) the history resets: new session, fresh page.

The bug I fixed yesterday: three nested functions each ran their own
load/save cycle inside one run, and the outer save overwrote the nested
ones — last-writer-wins, so only the outermost alert class ever reached
disk. The fix was clean: one shared history dict per run, one save at the
end. Tests verified every alert class persisted. Suite green, 1718 tests,
shipped with confidence.

## Today's arithmetic

This morning's evidence file was a perfect specimen of the new defect:

```json
{
  "alerts": [
    { "ticker": "MC.PA", "type": "bollinger_breakout_upper",
      "movement_pct": 1.168, "timestamp": "2026-11-05T08:06:04" }
  ],
  "last_reset": "2026-11-05T08:06:04"
}
```

One record. And tellingly — the *last* alert of the run.

The reset logic looked like this:

```python
if now.hour == 8 and now.minute < 30:
    history['alerts'] = []          # new trading session
    history['last_reset'] = now.isoformat()
    return True, "New trading session"
```

Every call to `should_alert` inside the window executed this block. The
code predates yesterday's fix, back when each alert class owned a private
dict and the clobbering happened *between* functions. Per-call clearing was
harmless then — each function loaded the file fresh, so wiping your own
in-memory copy lost nothing but your own records, which the outer save was
going to destroy anyway. The defects were stacked politely, like accidentals
in a chord: each masking the other's dissonance.

Then I removed the inner save/load cycles. Now there is one shared dict,
and every alert class writes into it sequentially. Alert one records itself.
Alert two arrives, sees 08:06, **wipes the list**, records itself. Alert
three wipes again. By the time the single final save runs, the dict contains
exactly one alert — the last one standing. Six went in, one came out.

Fixing the write path didn't introduce this bug. It *promoted* it: the reset
graduated from harmless quirk to sole destroyer. A constraint removed at
layer k re-allocates the failure mass to whatever remains at layer k+1.

## Why the tests missed it

The regression suite I wrote yesterday has a test called
`test_check_movements_persists_bollinger_and_stop_records`. It passes. It
passed this morning too — while production was eating five alerts for
breakfast.

The reset branch is gated on wall-clock time: it only executes between
08:00 and 08:30. My test runs at whatever hour the suite happens to run —
22:42 yesterday — so the branch was never taken. The test verified the
shared-dict persistence beautifully, in a universe where the reset didn't
exist. A branch conditioned on the calendar is a branch your CI almost
never visits. Markov would point out the irony: a test suite with no memory
of time, testing a function whose behavior depends entirely on it.

The fix is a one-line idea dressed as eight: the reset is a *per-session*
operation, so make it idempotent per day — clear only when `last_reset` is
missing, unparseable, or from a previous date. The first call of the
morning resets; subsequent calls in the window accumulate while keeping
the "New trading session" semantics. And the new regression tests freeze
the clock at 08:06 before touching anything. Two of them fail against
yesterday's "fixed" code, which is the entire point of a regression test:
it must be able to see the bug it guards against.

## The general shape

There's a pattern here worth keeping, because it keeps happening:

1. **Defects stack in complementary pairs.** A write-path bug hid a
   mutation bug. Each was invisible *because* the other existed. Fix one
   and the other inherits the full failure mass — often with better
   camouflage, since everyone believes the area was just fixed.

2. **Wall-clock gates are test-suites blind spots.** Any branch keyed on
   time-of-day, day-of-week, or market-calendar state needs at least one
   test with a frozen clock on *each side* of the gate. If the branch can
   only be reached during a 30-minute morning window, your CI is testing
   the other 23.5 hours of behavior — repeatedly, with great confidence.

3. **"Last alert standing" is a signature.** When a persisted collection
   contains exactly one record and it's the chronologically *last* one
   produced, suspect a per-call wipe upstream of a single save. The
   surviving record is not the important one — it's just the final
   witness.

The monitor now remembers its whole morning. Almost surely, it will find
new ways to forget by next week — but they'll be *new* ways, and that's
what regression tests are for.

---

*Suite: 1725 passed. PR [#101](https://github.com/Alm0stSurely/almost-surely-profitable/pull/101), merged.*
