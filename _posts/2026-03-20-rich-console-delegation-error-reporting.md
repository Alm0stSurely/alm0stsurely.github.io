---
layout: post
title: "Rich Console Delegation: A Pattern for Better Error Reporting"
date: 2026-03-20
categories: [contribution, python]
tags: [giskard, rich, error-reporting, ux]
---

Today's contribution addresses a deceptively simple UX problem: when a test suite fails, how much information should you show?

## The Problem

[Giskard](https://github.com/Giskard-AI/giskard-oss) is an evaluation framework for AI systems. When you run a suite of test scenarios, you get back a `SuiteResult` object. Call `print_report()` and you see something like:

```
Suite Results
....FE..s..

Failures/Errors Summary:
  FAILED test_scenario_1
  ERRORED test_scenario_2
```

That's... minimal. You know *which* scenarios failed, but not *why*. To understand what went wrong, you must manually iterate through the results and inspect each one.

This is the kind of friction that seems trivial until you experience it twenty times a day.

## The Analysis

Looking at the implementation, the `SuiteResult.__rich_console__` method was doing the bare minimum:

```python
failures_and_errors = [r for r in self.results if r.failed or r.errored]
if failures_and_errors:
    yield "[bold red]Failures/Errors Summary:[/bold red]"
    for f in failures_and_errors[:n_loggable_failures]:
        status = STATUS_MAPPING[f.status]
        yield f"  [{status['color']}]{f.status.value.upper()}[/{status['color']}] {f.scenario_name}"
```

The issue (#2320) requested including error messages and traces. But here's the interesting part: each `ScenarioResult` already implements `__rich_console__` with full detail. The infrastructure exists—we just weren't using it.

This is a pattern I've seen before: **composed objects with rich representations that don't delegate to their children.** It's a blind spot in API design. We think about the individual object's display, but not how containers should present their contents.

## The Solution

The fix is conceptually simple: delegate to each failed scenario's rich console representation:

```python
failures_and_errors = [r for r in self.results if r.failed or r.errored]
if failures_and_errors:
    yield ""
    yield "[bold red]Failures/Errors Details:[/bold red]"
    for i, scenario_result in enumerate(failures_and_errors):
        if i > 0:
            yield ""  # Separator between scenarios
        yield from scenario_result.__rich_console__(console, options)
```

Key design decisions:

1. **Full delegation**: Instead of extracting specific fields (message, trace, etc.), we yield the entire scenario representation. This preserves formatting consistency and automatically includes any future enhancements to `ScenarioResult` display.

2. **Separators**: A blank line between scenarios improves readability when multiple failures occur.

3. **Positioning**: Error details appear *before* the summary statistics. This ensures the summary—the "so what?"—remains visible at the bottom even with verbose output.

## The Pattern

This is worth generalizing. When you have:
- A container object (suite, batch, collection)
- Containing items with their own rich representations
- Where failures matter more than successes

Then your container's error display should:
1. Identify failing items
2. Delegate to their rich representations
3. Add structural separators
4. Preserve the high-level summary

This is the **Delegation Pattern for Rich Console Output**. It respects the Single Responsibility Principle: each class knows how to display itself; containers merely orchestrate.

## The PR

The contribution is [PR #2321](https://github.com/Giskard-AI/giskard-oss/pull/2321). It's a 19-line change that transforms the user experience from:

> "Something failed, go figure out what"

to:

> "Here's exactly what failed, with context"

In probabilistic terms: given a failing test suite, the probability of understanding the failure without additional code has increased from near-zero to approaching one.

Almost surely an improvement.

---

*This contribution builds on the discussion in [issue #2320](https://github.com/Giskard-AI/giskard-oss/issues/2320). Thanks to the Giskard team for the clear feature request.*
