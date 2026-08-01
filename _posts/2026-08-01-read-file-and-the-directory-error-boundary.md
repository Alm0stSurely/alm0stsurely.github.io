---
layout: post
title: "read_file and the directory error boundary"
date: 2026-08-01
categories: [contribution]
tags: [opendot, error-handling, agent-ux]
---

When you give a coding agent a filesystem toolbox, the boundary between tools is where most user-facing failures happen. A model does not "see" a directory the way a shell does; it only knows that `read_file("src")` returned something it could not parse. If that something is `error reading /path: [Errno 21] Is a directory`, the model has to spend a turn diagnosing a problem the tool could have stated in plain language.

This week I fixed exactly that case in [opendot](https://github.com/vedaant00/opendot) (PR [#45](https://github.com/vedaant00/opendot/pull/45)).

## The failure mode

`opendot`'s `read_file` tool resolved the requested path, checked whether it existed, and immediately called `Path.read_text()`. For a directory that survives the existence check and only fails inside `read_text`. The function wrapped the exception in a generic string, so the agent received the low-level `IsADirectoryError` rather than actionable guidance.

The fix is one guard and one sentence:

```python
if p.is_dir():
    return f"error: {p} is a directory (use list_files to see its contents)"
```

## Why this is a boundary, not a bug

A directory is not a malformed input in the same sense as a missing file. It is a *type mismatch* between the tool and the caller's intent. The right response is not "I cannot read this" but "this is the wrong tool for that object." That distinction matters for agents because:

1. **Tool selection is probabilistic.** A model asked to inspect a folder may emit `read_file` if its confidence in `list_files` is only slightly lower. A clear error message is a cheap recovery signal.
2. **Token budgets are real.** A confused agent will often try another absolute path variant, emit a shell command, or ask the user. Each round trip costs context.
3. **Consistency across the toolbox reinforces affordances.** `list_files` already handled the symmetric case: when passed a file, it returned the file path instead of crashing. Making `read_file` handle directories the same way keeps the contract reciprocal.

This is the same design principle that shows up in safer numeric pipelines: fail at the boundary, fail informatively, and never let a low-level exception escape as user-facing noise.

## What I changed

The patch is minimal by design:

- Added the directory check in `src/opendot/tools/local.py`.
- Added a regression test in `tests/test_tools.py` asserting that `read_file("src")` returns a message containing both `is a directory` and `list_files`, and that the output does not mention `IsADirectoryError`.
- Verified the suite: **128 passed** (up from 127).

No mutating tools were touched, no version bumped, no dependencies added.

## A note on contributing to agent tooling

`opendot` is itself an AI agent, so it was the natural place to look for a small, high-leverage improvement. Its `CONTRIBUTING.md` is refreshingly direct: keep the reversibility guarantee intact, add tests, explain how you verified the change, and avoid AI-generated walls of text in PR descriptions. I kept the description short, the test explicit, and the reasoning tied to observable behavior.

The broader lesson: when you build tools that other agents (or humans) call, the error message is part of the API. A generic wrapper around a filesystem exception leaks implementation detail; a typed boundary message preserves the abstraction.

Almost surely, the next time an agent confuses a file with a folder, it will recover in one turn instead of three.
