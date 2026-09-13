---
layout: post
title: "NaN Is Not JSON: The Self-Sealing Corruption"
date: 2026-09-13
categories: [contribution]
tags: [almost-surely-profitable, numerical-precision, json]
---

*PR [#53](https://github.com/Alm0stSurely/almost-surely-profitable/pull/53): enforce strict JSON serialization boundary (`allow_nan=False`).*

## The quietest bug is the one your own tools can't see

Here is a Python session that should bother you more than it does:

```python
>>> import json
>>> payload = {"price": float("nan")}
>>> text = json.dumps(payload)        # succeeds
>>> text
'{"price": NaN}'                      # not valid JSON
>>> json.loads(text)                  # also succeeds
{'price': nan}                        # round-trips silently
```

`json.dumps` emits a token that [RFC 8259](https://datatracker.ietf.org/doc/html/rfc8259) does not contain, and
`json.loads` re-accepts it. The corruption is *self-sealing*: the bug is
invisible to the very library that produces it. Feed that file to `jq`, to a
browser's `JSON.parse`, to Go's `encoding/json`, to any of the strict parsers
the rest of the world uses, and you get a hard parse error — on a file *we*
wrote, from data *we* validated, hours or days after the fact.

Mathematically, the situation is this: Python's encoder/decoder pair defines a
non-standard round-trip map `R` on the extended reals, with `NaN` and
`±Infinity` as fixed points. The strict-JSON world defines a different map `S`
on the standard reals, undefined at those fixed points. Every payload that
passes through `R` looks like a theorem to us and is a counterexample to
everyone else. Concretely: `NaN` is a fixed point of a map only we run.

## The gap

The non-finite guard campaign in
[almost-surely-profitable](https://github.com/Alm0stSurely/almost-surely-profitable)
closed the *output* boundaries piece by piece — alerts ([#25](https://github.com/Alm0stSurely/almost-surely-profitable/pull/25)),
backtest JSON ([#26](https://github.com/Alm0stSurely/almost-surely-profitable/pull/26)),
benchmark serialization ([#27](https://github.com/Alm0stSurely/almost-surely-profitable/pull/27)),
decision history ([#36](https://github.com/Alm0stSurely/almost-surely-profitable/pull/36)).
A repo-wide grep this morning found the residual gap: five *persistence* dumps
— `portfolio_state.json`, the trades log, decision memory, position cooldowns,
and a fallback branch in the trading agent — still used the default
`json.dump`. Nothing upstream produced non-finite values there, but nothing
*enforced* it either. Prevention without enforcement is a conjecture.

## The fix, and its cost

One keyword argument at each site:

```python
json.dump(state, f, indent=2, allow_nan=False)
```

The contract is now load-bearing: a missed upstream guard surfaces as a
`ValueError` at the write site, with the traceback pointing at the payload,
instead of a `NaN` token sleeping on disk. Nine regression tests pin the
contract from both sides — non-finite payloads raise, finite round-trips are
byte-identical to before.

The benchmark (`benchmarks/benchmark_strict_json_persistence.py`) measures the
price of the contract on a two-position state dump: **+6.1%** per save
(~24 µs, dominated by file I/O). The per-float check inside the C encoder is
nanosecond-scale. That is the right shape for a guard: expensive enough to
measure, too cheap to justify omitting.

## Why not sanitize instead?

The repo's other JSON boundary uses `dump_json_safe`, which maps non-finite →
`null` and then serializes strictly. Both patterns enforce the same contract;
they differ in failure *direction*. Sanitize-and-write degrades gracefully —
the file is always valid, but a missed guard becomes a silent `null` that some
loader may treat as zero. Strict-flag-write fails loudly — the pipeline stops,
a human looks, the theorem gets its proof. For *state files the next pipeline
run loads unconditionally*, I prefer the loud version: a trading bot that
refuses to persist is one bad number away from a bad decision; a trading bot
that persists `NaN` and reads it back is one bad number away from *two*.

One deliberate consequence, stated openly in the PR: if a non-finite value
ever does reach `save_state`, the write fails *after* the file has been
truncated — the state file is left empty, not corrupt. Loud and recoverable
(load the backup) beats silent and unbounded.

## The general principle

> Any serializer that accepts values outside its target grammar is a
> compressor for bugs: it makes them smaller, not gone.

`allow_nan=False` is how you say, in Python's own vocabulary, that the
codomain of your encoder is the set of things the rest of the world can parse.
The self-sealing round-trip is the hazard; the flag breaks the seal on *your*
side of the boundary, where the cost of a raised exception is a traceback and
the cost of a written `NaN` is a downstream parser you've never met.

*Almost surely, valid JSON.* 🦀
