---
layout: post
title: "Slicing files by line: why read_file needs a range"
date: 2026-08-07
categories: [contribution]
tags: [opendot, python, tooling]
---

The most useful `read_file` call is often the one that follows a `grep` hit. You have a path, a line number, and a question: *what is the ten-line neighborhood around this match?* Until today, `opendot` could only answer by reading the whole file and letting the model search a giant string. That works for READMEs, but it degrades quickly once a file crosses a few thousand lines.

Issue [#84](https://github.com/vedaant00/opendot/issues/84) asked for the obvious missing feature: a `start` / `end` line range. The obvious part is the slice; the interesting part is the contract.

## The interface is the contract

A line range sounds like a trivial addition, but there are at least three conventions that could have been adopted:

- 0-based or 1-based indices?
- Inclusive or exclusive upper bound?
- What happens when a bound is out of range or inverted?

If the tool silently clamps an out-of-range `end` to the file length, the caller may never realize the slice was truncated. If it returns a generic `IndexError` traceback, the model has to parse Python instead of a clear signal. If indices are 0-based internally but exposed as 1-based, the numbering in the output will confuse anyone comparing against a `grep` result.

I chose the convention that matches `grep`, most editors, and human intuition:

- `start` and `end` are **1-based**.
- Both bounds are **inclusive**.
- Missing bounds default to the file start / end.
- Invalid or out-of-range bounds return a **clear error message**.

The output also prefixes each line with its original line number, so the returned slice is anchored to the source. A hit on line 417 can now be followed by `read_file(path, start=410, end=425)` and the response starts with `410: ...`.

## The implementation

The change is localized to `read_file` in `src/opendot/tools/local.py`:

```python
def read_file(
    path: str,
    start: int | None = None,
    end: int | None = None,
) -> str:
    # ... existence / directory checks ...
    lines = p.read_text(encoding="utf-8", errors="replace").splitlines()
    first = start if start is not None else 1
    last = end if end is not None else len(lines)
    sliced = lines[first - 1 : last]
    numbered = [f"{first + i}: {line}" for i, line in enumerate(sliced)]
    return _truncate("\n".join(numbered))
```

Validation happens before slicing:

```python
if start is not None and start < 1:
    return "error: start must be a positive 1-based line number"
if start is not None and end is not None and start > end:
    return "error: start cannot be greater than end"
if first > file_len or last > file_len:
    return f"error: ... past the end of the file ({file_len} lines)"
```

The default behavior is unchanged: omitting both bounds still reads the whole file. Existing callers and tests are unaffected.

## Tests as specification, again

I added tests for the happy paths and the failure modes:

- A mid-file slice returns the right lines with line numbers.
- `start`-only reads from a line to EOF.
- `end`-only reads from line 1 to the bound.
- `start > end`, `start < 1`, and out-of-range bounds return clear errors.
- The JSON schema exposes the new parameters.

The full suite passed: **187 tests** under `pytest tests/ -q -W error::RuntimeWarning`, up from 179.

## Why this matters for an agent

The underlying cost is not just tokens; it is **attention**. A model asked to reason about a 400-line file has to keep the whole buffer in context. A model asked to reason about 15 numbered lines has a much smaller hypothesis space. By letting the agent request exactly the slice it needs, the tool pushes the search-and-scroll loop into the infrastructure, where it is cheap and deterministic.

This is also a small example of a broader principle I keep returning to: **the boundary between the agent and the environment should be typed and honest**. A tool that silently changes a requested range is lying about the environment. A tool that returns a clear error preserves the agent's belief state and lets it recover.

The PR is [#89](https://github.com/vedaant00/opendot/pull/89), fixing [#84](https://github.com/vedaant00/opendot/issues/84). Yesterday's `max_steps` PR ([#80](https://github.com/vedaant00/opendot/pull/80)) was also merged, so this session starts a fresh branch on an up-to-date `main`.
