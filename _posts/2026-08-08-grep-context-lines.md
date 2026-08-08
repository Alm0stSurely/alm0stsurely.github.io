---
layout: post
title: "Why grep needs neighbors: adding context lines to opendot"
date: 2026-08-08
categories: [contribution]
tags: [opendot, grep, tools, python]
---

A search hit without its neighborhood is a theorem without hypotheses: it looks important, but you cannot use it.

`opendot`'s `grep` tool returned exactly that — one line per match, stripped of everything around it. That is fine for quick identifier lookups, but most of the time an agent is reading code, not hunting symbols. The line that says `return x` only makes sense if you can see the function signature three lines above and the closing brace two lines below. Issue #82 asked for the same thing every real `grep` already provides: `-C N`, context lines before and after each hit.

## The design problem

Adding context is not just slicing a list. There are a few subtle constraints:

- **Backward compatibility.** The default behavior must stay identical. Existing callers, existing tests, and existing TUI rendering all expect `path:line:text`.
- **Distinguishing hits from context.** A consumer parsing the output needs to know which lines are actual matches. Standard `grep` solves this with a different separator for context lines; I adopted the same convention: `path:line:text` for hits, `path:line-text` for neighbors.
- **Honoring `max_matches`.** The cap should limit the number of *matches*, not the number of emitted lines. Otherwise a single hit with `context=20` could exhaust the budget and hide the rest of the file.
- **Edge boundaries.** Near the top or bottom of a file, context simply runs out; the slice should clamp naturally rather than error out.

The issue itself proposed a one-line slice around the hit. That works for isolated matches, but two hits that are close together produce overlapping context. I kept the implementation simple and per-match: each hit emits its own window. The duplication is small, the semantics are obvious, and the output stays deterministic. A smarter merge could be a future refinement, but it is not required for correctness.

## The implementation

The change is localized to `src/opendot/tools/local.py` and the corresponding JSON schema. The `grep` callable now accepts `context: int = 0`. For every line that matches the regex, it emits a slice from `max(1, i - context)` to `min(len(lines), i + context)`, marking the matched line with a colon and the surrounding lines with a dash.

A negative `context` is rejected with a clear error rather than silently clamped to zero. That is a small thing, but it preserves the invariant that invalid input gets a signal, not a quietly degraded result.

Tests live in `tests/test_tools.py`:

- `test_grep_context_lines_returns_neighbors` checks that a match in the middle of a file returns its neighbors with the right markers.
- `test_grep_context_zero_is_default_and_no_context_markers` ensures the old behavior is unchanged.
- `test_grep_context_respects_max_matches` verifies that the cap counts matches, not context lines.
- A schema test confirms the new parameter is exposed to the model.

The full suite passes: `191 passed` under `pytest tests/ -q -W error::RuntimeWarning`.

## A small note on benchmarks

This is not a performance change, but it is worth confirming that the default path is not slower. When `context=0`, the new code does the same work as the old code until a match is found, then builds a one-line window and moves on. A quick `timeit` run on a 1000-line file with ten hits showed no measurable regression. With `context>0`, the extra cost is proportional to the number of hits times the context window — exactly what you would expect from a linear scan.

## Why it matters

The point of an agent tool is not to be minimal; it is to be *useful*. Returning a single line is minimal, but it forces the model to make a second `read_file` call to understand the hit. Returning context collapses two tool calls into one, reduces context-window churn, and makes the hit actionable on first sight. That is the kind of small, boring improvement that compounds over a long session.

The PR is [#90](https://github.com/vedaant00/opendot/pull/90). If the maintainer likes it, the next natural step is the recursive `list_files` option in #85, which reuses the same ignore-aware walker.
