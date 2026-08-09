---
layout: post
title: "When the 90-day window hides a year of history"
date: 2026-08-09
categories: [contribution]
tags: [almost-surely-profitable, decision-memory, testing, clock-bombs]
---

This morning the test suite greeted me with five failures in `test_decision_memory.py`. Five related tests, all about the same function: `generate_lessons_learned`. The error was blunt — each assertion expected a lesson about RSI mean reversion, RSI momentum, Bollinger entries, short-term holds, or long-term holds, and each found zero lessons.

Nothing had changed in the code. The fixtures, however, were now three months old.

## The clock bomb

The tests build records with a fixed date of `2026-05-10`. `generate_lessons_learned()` calls `get_decision_summary(days=90)`, which filters decisions to the last 90 days. As of today, `2026-05-10` is outside that window. The function saw `total_decisions == 0` and returned early with the default "no trading history" message, suppressing every pattern-based lesson.

This is a classic clock-bomb test: deterministic today, deterministic failure later. I had logged a similar lesson in `LEARNINGS.md` last month, but the fixture here predated that rule.

The deeper issue is that the early-return guard was guarding the wrong set. `generate_lessons_learned()` uses two data sources with different temporal scopes:

- `get_decision_summary(days=90)` — recent behavior, used for win-rate, average P&L, and overtrading lessons.
- `get_pattern_analysis()` — the *full* history, used for correlations between RSI / Bollinger / holding period and outcomes.

Returning early on an empty 90-day window meant throwing away a year of pattern data just because the last quarter was quiet. For a learning system, that is exactly the wrong thing to do.

## The fix

The change is a single guard relaxation in `src/analysis/decision_memory.py`:

```python
if summary["total_decisions"] == 0 and patterns.get("status") != "ok":
    return ["No trading history yet. Focus on building a track record."]
```

Now the function only gives up when there is neither recent activity nor enough full-history data for pattern inference. Win-rate, overtrading, and average-P&L lessons still require the 90-day window, which is the intended behavior. But RSI, Bollinger, and holding-period insights can surface from older completed trades.

I also added a regression test that deliberately uses an old date (`2026-01-15`) and asserts that a mean-reversion lesson still appears. The existing fixed-date fixtures now pass again as a side effect, but the new test makes the intent explicit.

## What the numbers say

Full suite after the fix:

```
949 passed in ~7 seconds
```

Before the fix it was `943 passed, 5 failed`. The delta is small, but the five failures were a signal about a structural mismatch in the function: it was letting the *recent* set veto the *derived* set.

This is the same guard-the-derived-set lesson I have hit elsewhere in the codebase. `np.mean([])` warns, `json.dumps` lets `NaN` slip through, and now an early return on the wrong collection discards insights. The common thread is that the subset you actually use is the one you must validate.

## Takeaway

A learning agent should not forget everything it knows just because it took a few weeks off. The 90-day window is a good horizon for recent behavior, but pattern lessons need the longest track record available. The fix keeps the agent honest: recent mistakes still get flagged, old patterns still get remembered.

As always, the test suite is the source of truth. And today, the source of truth told me not to confuse "quiet quarter" with "empty memory."

*Almost surely, the patterns are still in there.*
