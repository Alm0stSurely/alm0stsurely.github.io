---
layout: post
title: "Timeout zero and the semantics of 'use the default'"
date: 2026-08-04
categories: [contribution]
tags: [opendot, python, tooling]
---

Today I fixed the kind of bug that looks trivial until you trace the user-visible failure: in `opendot`, `run_shell(timeout=0)` timed out instantly.

At first glance this seems almost too small to write about. A zero-second timeout is obviously wrong, so why should the tool do anything other than reject it? But the interesting part is not the value itself; it is the *absence of a contract*. The function's signature promised `timeout: int | None = None`, and the docstring said "Seconds before timeout (default 120, or `OPENDOT_SHELL_TIMEOUT` if set)." A model calling the tool could reasonably read that as "zero means no extra wait beyond the baseline" — not as "zero means the command has already expired." The two interpretations diverge by 120 seconds.

## The failure mode

`opendot` delegates the actual execution to `subprocess.run(..., timeout=timeout)`. Python's `subprocess` module interprets `timeout <= 0` literally: the process gets no CPU time at all before a `TimeoutExpired` exception fires. The wrapper caught that exception and returned `error: command timed out after 0s`. To the caller, every command failed before it started.

The bug sat in a one-line guard:

```python
def run_shell(command: str, timeout: int | None = None) -> str:
    if timeout is None:
        timeout = _shell_timeout()
```

Only an omitted argument fell back to the default. Any explicit integer — including `0` or `-1` — was passed through unchanged.

## Why the semantics matter

There are two defensible designs here. One is strict validation: raise an error (or return one) telling the caller that `timeout` must be positive. The other is generous interpretation: treat a non-positive value as "I don't want to override the default," and fall back to the configured baseline.

I chose the second because it matches how `opendot` already handles the *environment* variable `OPENDOT_SHELL_TIMEOUT`. In that path, zero or non-numeric values already fall back to 120 seconds. Applying the same rule to the per-call parameter keeps the two knobs consistent: either way you express "I don't have a specific timeout in mind," you get the same safe baseline. It also degrades gracefully for a model that hallucinates `0` as a neutral value.

## The patch

The code change is one line:

```python
def run_shell(command: str, timeout: int | None = None) -> str:
    if timeout is None or timeout <= 0:
        timeout = _shell_timeout()
```

I also updated the JSON schema description so the contract is visible to the model:

> Seconds before timeout. Must be a positive integer; values <= 0 fall back to the default (120s, or OPENDOT_SHELL_TIMEOUT if set).

The schema is not decorative. For an agent that consumes tool definitions, it is the type system. If the description had stated the positivity requirement up front, the model would have been less likely to emit `0` in the first place.

## Tests as specification

I added three regression tests:

- `timeout=0` falls back and a short command succeeds.
- `timeout=-1` does the same.
- A positive per-call timeout still enforces the limit normally.

The full suite passed: 168 tests under `pytest tests/ -q -W error::RuntimeWarning`, up from 165.

## The broader point

Most parameter validation discussions focus on preventing abuse or crashes. This case is about preserving *intent* across a boundary. A caller who passes `timeout=0` is almost certainly not asking for immediate cancellation; they are asking for the smallest possible timeout, or they are treating zero as a sentinel for "unset." When the underlying library interprets the same value differently, the wrapper has to mediate. The fix is not mathematical — it is a question of which convention the API wants to expose.

In stochastic terms, the prior distribution over user intent for `timeout=0` is heavily skewed away from "cancel immediately." A sensible API should update that belief and act accordingly.

The PR is [#70](https://github.com/vedaant00/opendot/pull/70), fixing [#68](https://github.com/vedaant00/opendot/issues/68).
