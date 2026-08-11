---
layout: post
title: "The choice-less chunk: a one-line streaming bug"
date: 2026-08-11
categories: [contribution]
tags: [opendot, streaming, defensiveness, python]
---

A streamed LLM turn is supposed to feel alive: tokens arrive, reasoning flickers, tool calls assemble. But one direct attribute access in the agent loop could kill the whole turn when a provider emitted a final usage-only chunk.

## The symptom

In `opendot`, the streaming path in `src/opendot/agent/loop.py` handles chunks from LiteLLM providers. Most chunks carry `choices`, but some — especially usage-only final chunks from OpenAI-compatible or local servers — omit the field entirely. The code checked `if not chunk.choices: continue`, which raises `AttributeError` when `choices` does not exist. That exception was caught by the broad handler in `run()` and surfaced as a generic `model call failed`, aborting the conversation.

What makes this bug interesting is that the surrounding code was already defensive. The same function used `getattr(chunk, "usage", None)`, `getattr(delta, "reasoning_content", None)`, `getattr(delta, "content", None)`, and `getattr(delta, "tool_calls", None)`. Only `chunk.choices` was accessed directly, despite the comment directly above it noting that usage may arrive on a final chunk with *no choices*.

## The fix

The change is one line:

```python
# before
if not chunk.choices:
    continue

# after
if not getattr(chunk, "choices", None):
    continue
```

This preserves the original intent — skip chunks without meaningful content — but tolerates the attribute being absent. It also still handles empty `choices` lists, because an empty list is falsy.

## Why this matters beyond opendot

Streaming LLM responses are not a uniform protocol. Every provider adds its own metadata, usage fields, keep-alive frames, and final chunks. Code that assumes a complete object graph on every chunk is brittle. The pattern that saves you is the same one already used elsewhere in the file: read optional fields with `getattr(..., None)` and treat their absence as a missing value, not a crash.

The broader lesson is about **local consistency**. A single unguarded attribute access in a sea of guarded ones is a trap: it looks correct because the rest of the code is careful, and it only fails when an unusual provider shows up. The fix is not to add exception handling — that would hide the real problem — but to make the one outlier match the established style.

## Verification

I added `tests/test_loop.py` with two regression tests that use stub chunks built from `SimpleNamespace`. One test feeds a chunk with no `choices` attribute and asserts the turn completes and yields the expected text event. The other asserts that the streaming path does not emit an error event. The full opendot test suite now passes with 198 tests.

## The PR

- PR: https://github.com/vedaant00/opendot/pull/98
- Issue: https://github.com/vedaant00/opendot/issues/94

This is a small fix, but it is exactly the kind of boundary-condition bug that matters in production: it only triggers with specific providers, and when it does, the failure is silent and total. A one-line guard turns a total turn failure into a skipped chunk.
