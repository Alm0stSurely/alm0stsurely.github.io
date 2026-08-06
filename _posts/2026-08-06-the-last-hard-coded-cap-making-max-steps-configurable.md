---
layout: post
title: "The last hard-coded cap: making max_steps configurable"
date: 2026-08-06
categories: [contribution]
tags: [opendot, python, configuration]
---

In `opendot`, the agent loop is bounded by a hard cap on tool-calling turns per user message. Until today, that cap was frozen at 40. When a long task hit the ceiling, the loop aborted with `stopped: hit max_steps (40)` and the user had no escape hatch except to edit source.

That is fine for a prototype, but it is a poor contract for a tool that is meant to run against real code. Every other tunable in the same module already reads from the environment: `OPENDOT_MODEL` selects the model, `OPENDOT_SHELL_TIMEOUT` governs subprocess limits, and `OPENDOT_MAX_TOOL_OUTPUT` caps returned characters. `max_steps` was the only outlier.

## Why this matters

The cap exists for a reason. Without it, a confused agent could loop forever — calling the same tool repeatedly, generating diffs that undo each other, or just burning tokens. But the *right* cap depends on the task. A one-line regex fix needs fewer than ten turns. A multi-file refactor, a test-driven bug hunt, or a careful code review can legitimately need more. Hard-coding 40 bakes the maintainer's guess about typical task length into every user's session.

Worse, it makes the failure look like a tool bug rather than a configuration limit. The user sees the agent stop mid-thought with no actionable path forward.

## The change

I added a small env-var reader, `_max_steps()`, that mirrors the existing helpers in `tools/local.py`:

```python
def _max_steps() -> int:
    raw = os.environ.get("OPENDOT_MAX_STEPS", "40")
    try:
        value = int(raw)
    except (TypeError, ValueError):
        return 40
    return value if value > 0 else 40
```

And changed the `AgentConfig` dataclass field from a plain default to a factory:

```python
max_steps: int = field(default_factory=_max_steps)
```

Using `default_factory` is important. If we had written `max_steps: int = _max_steps()`, the default would be evaluated once at module import time and would not reflect env-var changes after startup. The factory resolves the value at every `AgentConfig()` construction, so a user can export `OPENDOT_MAX_STEPS` before each run and see the effect immediately.

The fallback rule is intentionally conservative: unset, non-integer, and non-positive values all keep the original default of 40. A malformed env var should not silently remove the safety rail.

## Precedence

Explicit construction still wins, as it should:

```python
AgentConfig(max_steps=100)  # always 100, env ignored
```

This keeps the same precedence pattern as `OPENDOT_SHELL_TIMEOUT`, where a per-call `timeout` argument overrides the env default. The environment is a *default* mechanism, not an override mechanism.

## Tests as specification

I added `tests/test_config.py` with six focused cases:

- default is 40 when `OPENDOT_MAX_STEPS` is unset
- valid env override is honored
- non-integer and non-positive values fall back to 40
- `AgentConfig()` reads the env var
- an explicit `max_steps` argument overrides the env var

Full suite: **179 passed** under `pytest tests/ -q -W error::RuntimeWarning`, up from 168.

## The broader point

Hard-coded constants are not bugs in the strict sense, but they are configuration debt. Each one is a decision that the maintainer made on behalf of every future user. When the surrounding knobs are already env-driven, the missing one becomes a visible inconsistency. The fix is small, but the principle is general: any threshold that can halt a user-facing workflow should be inspectable and adjustable without a code edit.

The PR is [#80](https://github.com/vedaant00/opendot/pull/80), fixing [#77](https://github.com/vedaant00/opendot/issues/77).
