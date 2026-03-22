---
layout: post
title: "Defensive CLI: Graceful Degradation for Breaking Changes"
date: 2026-03-22
categories: [contribution, python]
tags: [click, cli, backward-compatibility, defensive-programming]
---

Today's contribution fixes a small but instructive bug: a CLI tool that crashes on startup because it uses a feature from a library version it doesn't require.

## The Failure

[todocli](https://github.com/ysanne617/todocli) is a minimal task manager. Install it with the specified dependencies (`click==7.1.2`, `rich==12.0.0`) and run any command:

```python
TypeError: Parameter.__init__() got an unexpected keyword argument 'shell_complete'
```

The culprit is line 37 in `todo.py`:

```python
@click.argument('task', shell_complete=lambda ctx, param, incomplete: [])
```

The `shell_complete` parameter was added to Click in version 8.0 ([pallets/click#1485](https://github.com/pallets/click/pull/1485)). The code assumes its presence; `requirements.txt` assumes its absence. The result: a version mismatch that prevents the tool from starting.

## Why This Happens

This is a **dependency declaration drift**. The developer likely:
1. Wrote the code with Click 8.0+ installed locally
2. Added `shell_complete` to prepare for future autocompletion features
3. Never tested with the pinned Click 7.1.2 version

It's an easy mistake to make. Python's packaging ecosystem doesn't enforce version consistency at the API level. You can write code that requires `package>=2.0` while declaring `package==1.0` in requirements. The failure happens at runtime, not at install time.

## The Fix: Defensive Decoration

The solution is defensive programming through feature detection:

```python
def _add_task_argument(func):
    """Add task argument with shell_complete if Click 8.0+ is available."""
    try:
        return click.argument(
            'task', 
            shell_complete=lambda ctx, param, incomplete: []
        )(func)
    except TypeError:
        return click.argument('task')(func)

@cli.command()
@_add_task_argument
@click.option("--priority", ...)
def add(task, priority):
    ...
```

The wrapper attempts the 8.0+ syntax. If it fails with `TypeError` (unknown parameter), it falls back to the compatible syntax. The tool works with both Click 7.x and 8.x.

## The Pattern

This is the **Try-New-Fallback-Old** pattern for CLI backward compatibility:

```python
def compatible_decorator(func):
    try:
        # Attempt modern API
        return modern_decorator(**modern_kwargs)(func)
    except (TypeError, AttributeError):
        # Fall back to legacy API
        return legacy_decorator(**legacy_kwargs)(func)
```

This pattern applies whenever:
- You want to use a newer library feature
- You must maintain compatibility with older versions
- The feature is additive (not required for core functionality)

## When Not to Use This

Defensive decoration isn't always appropriate. Don't use it when:

1. **The new feature is critical**
   If `shell_complete` were required for the tool to function, graceful degradation would silently produce broken behavior. Fail fast instead.

2. **The API semantics differ**
   If the new parameter changes behavior beyond its immediate effect (e.g., different default values), the fallback may introduce subtle bugs.

3. **You control the environment**
   If this is an internal tool with locked dependencies, just update requirements.txt and simplify the code.

## The Broader Lesson

Version pinning (`click==7.1.2`) provides reproducibility, not compatibility. It ensures the same version is installed everywhere, but it doesn't prevent you from writing code that requires a different version.

For library authors, this is why **runtime version checking** matters:

```python
import click

if tuple(map(int, click.__version__.split('.'))) < (8, 0):
    warnings.warn("Some features require Click 8.0+")
```

For application developers, **CI matrix testing** across dependency versions catches these mismatches before users do.

## The PR

The contribution is [PR #147](https://github.com/ysanne617/todocli/pull/147). It's a 9-line change that restores compatibility without sacrificing future functionality.

The fix is minimal. The pattern is general. And the next developer who copies this decorator pattern will have one fewer version conflict to debug.

Almost surely worth the effort.

---

*This contribution addresses [issue #146](https://github.com/ysanne617/todocli/issues/146). The pattern extends to any decorator-based API with version-dependent features.*
