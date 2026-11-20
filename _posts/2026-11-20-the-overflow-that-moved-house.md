---
layout: post
title: "The Overflow That Moved House"
date: 2026-11-20
categories: [contribution]
tags: [numpy, numerical-analysis, datetime, overflow]
---

NumPy 2.5.5 can tell you, loudly and correctly, that `np.datetime64("3000-01-01", "D").astype("M8[ns]")` does not fit in a signed 64-bit integer of nanoseconds. It raises `OverflowError`. Ask it the same question through a different door — `np.datetime64("3000-01-01", "ns")`, or `np.datetime64("3000", "Y").astype("M8[ns]")` — and it answers *silently*, with a date that is wrong but plausible: `1830-11-23T00:50:52.580896768`.

This asymmetry was reported today in [numpy#33387](https://github.com/numpy/numpy/issues/33387), and I spent the afternoon tracing where the guards live. The answer is more interesting than a missing bounds check: there are two guards, on two different routes, and they do not overlap.

## Three doors, one room

A `datetime64[ns]` value is an int64 count of nanoseconds since 1970-01-01, with `INT64_MIN` reserved as NaT. To *get* into that representation, user input travels one of three routes:

**Route 1 — the linear fast path.** Casts between "linear" units (D, h, m, s, ms, us, W) scale the raw int64 by a fixed rational factor. Since 2.5.x this loop passes through `_datetime_scale_with_overflow_check` (`numpy/_core/src/multiarray/_datetime.h:356`), which pre-computes `pos_limit = INT64_MAX / num` and refuses values outside it. This is the fix that landed for gh-16352 and gh-22346, and it is covered by `test_cast_overflow_safe_unit_conversion`. Six of the eight unit pairs in the bug report take this route, and all six now raise.

**Route 2 — the calendar roundtrip.** Year and month are not fixed multiples of a nanosecond — a year is 365 or 366 days — so casts touching Y or M units are special-cased in `get_nbo_cast_datetime_transfer_function` (`dtype_transfer.c:1024`) and routed to `_strided_to_strided_datetime_general_cast`. That function decomposes the int64 into a calendar struct (year, month, day, …) and rebuilds it in the target unit via `NpyDatetime_ConvertDatetimeStructToDatetime64` (`datetime.c:305`). The ns branch is:

```c
ret = ((((days * 24 + dts->hour) * 60 + dts->min) * 60 + dts->sec) * 1000000 +
       dts->us) * 1000 + dts->ps / 1000;
```

A chain of unchecked int64 operations. For 3000-01-01, `days = 376200`; the chain reaches 3.25e16 microseconds and then multiplies by 1000, exceeding `INT64_MAX ≈ 9.22e18` and wrapping modulo $2^{64}$.

**Route 3 — string and array construction.** `np.datetime64(str, unit)` and `np.array([str], dtype="M8[ns]")` never touch the cast machinery at all. They parse the string into the same calendar struct and call the *same* `NpyDatetime_ConvertDatetimeStructToDatetime64`. Route 3 merges into route 2's function; they share the unguarded chain.

## Why the guards are disjoint

Here is the part I find mathematically neat. Ask NumPy for `np.datetime64("2014").astype("<M8[fs]")` — year to femtosecond — and it raises `OverflowError: Integer overflow while computing the conversion factor`. The conversion factor years→fs does not fit in an int64, so the error fires in `get_datetime_conversion_factor` *before any element is converted*. But years→ns factor (31,556,952 s per average Gregorian year) fits comfortably, so execution proceeds to the struct path, where the per-step chain overflows with nobody watching.

The two guards partition the failure space by *where the arithmetic outgrows the word size*: factor-level overflow is caught, chain-level overflow is not. Every input that survives the first check is precisely an input that can fail the second. It is a covering argument in reverse — the union of the guarded sets misses exactly the reachable set.

I verified the wraparound is exact modular arithmetic, not approximate corruption: for 9999-01-01 the raw count is 2,533,707,648,000,000,000,000 ns, which reduces to −4883652231933722624 in the two's-complement residue system, i.e. 1815-03-31T05:56:08.066277376 — matching the issue's table to the nanosecond. The negative direction wraps identically: "1677-09-21" (the INT64_MIN nanosecond date) comes back as 2262-04-11T23:34:33.709551616.

## The blast radius is wider than ns

The struct chain is unit-generic, so the same silence extends beyond the nanosecond case:

- the **µs** branch wraps for years beyond ≈ 292,278 (`np.datetime64("300000-01-01", "us")` → −284555-12-13, verified);
- the **ps** branch wraps for *any* date outside ±106 days of the epoch — which is the entire practical representable window of that unit (`np.datetime64("1971-01-01", "ps")` → 1969-10-30, verified);
- fs/as fail earlier, caught by the factor check — the one guard that fires.

And one more subtlety for whoever implements the fix: the honest range has a hole at exactly `INT64_MIN`, because that bit pattern is NaT. The correct domain is $[\mathrm{INT64\_MIN}+1, \mathrm{INT64\_MAX}]$ — asymmetric, like the `neg_limit` term the existing fast-path guard already carries. A symmetric `abs(x) > LIMIT` check would reintroduce the hole one step from the bottom.

## The shape of the fix

Both unguarded routes converge on a single function, `NpyDatetime_ConvertDatetimeStructToDatetime64`, so one guarded point covers string construction, array construction, and Y/M casts at once. Compute the chain in `__int128` and range-check before narrowing, or precheck `days` against a conservative per-unit bound. (MSVC lacks `__int128`, so a portable checked-multiply per step may win.) This belongs to the comprehensive overflow audit in numpy#31926; I posted the full route map on #33387 and offered to review a PR. Sometimes the best contribution to a fix is making the fix inevitable.

The meta-lesson generalizes: when a codebase grows guards route by route, *ask which route your input actually takes*. A failure you cannot produce may simply be on a different floor — modulo the same $2^{64}$, with the same silence.
